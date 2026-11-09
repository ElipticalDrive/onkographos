# Multi-Mode Rendering with Iso-surfaces and Meshes

Also called: Hybrid Ray Tracing for Iso-surfaces and Meshes, Surface-Volume Hybrid Rendering

## Overview
This method renders scenes containing both volumetric data and explicit surface geometry (triangle meshes or implicit iso-surfaces) by combining hardware-accelerated ray-triangle intersection with volume ray marching. It solves the problem of correctly occluding volume samples with surface geometry, ensuring that rays do not integrate volume data behind opaque or semi-transparent surfaces. It differs from pure volume rendering by prioritizing surface intersection tests to clip the integration range, and from pure mesh rendering by retaining the ability to render internal volume structures up to the surface boundary.

## How it works
**Inputs and Outputs:**
- Input: A scene containing volume data (structured AMR or uniform grids), triangle meshes, and optional implicit iso-surfaces defined by scalar fields and threshold values.
- Input: Camera parameters (origin, direction, near/far planes).
- Input: Clipping plane definitions.
- Output: A 2D image with color and opacity values per pixel.

**Data Structures:**
1.  **Volume BVH:** A bounding volume hierarchy built over the volume data bricks/regions. Contains spatial information about cells and references scalar fields.
2.  **Iso-surface BVH:** A bounding volume hierarchy built over the implicit iso-surface geometry. Rebuilt when the iso-value changes.
3.  **Mesh BVH:** A bounding volume hierarchy built over the triangle mesh geometry.
4.  **Scalar Fields:** One or more scalar fields associated with the volume data, used for transfer functions and iso-surface extraction.

**Algorithm Steps:**
1.  **Ray Initialization:** For each pixel, generate a primary ray $R$ with origin $O$ and direction $D$. Initialize ray interval $[t_{min}, t_{max}]$ based on camera near/far planes and any active clipping planes. Clipping planes are implemented by intersecting the ray with the plane equation and updating $t_{min}$ or $t_{max}$ accordingly.
2.  **Mesh Intersection:** Trace ray $R$ against the Mesh BVH using hardware-accelerated ray-triangle intersection. Find the nearest hit point $P_{mesh}$ at distance $t_{mesh}$. If a hit exists and $t_{mesh} < t_{max}$, record $t_{mesh}$.
3.  **Iso-surface Intersection:** Transform ray $R$ into the volume space (coordinate system of the volume data). Trace the transformed ray against the Iso-surface BVH. Find the nearest hit point $P_{iso}$ at distance $t_{iso}$ in volume space. Transform $t_{iso}$ back to world space if necessary. If a hit exists and $t_{iso} < t_{max}$, record $t_{iso}$.
4.  **Ray Shortening:** Determine the nearest surface hit distance $t_{surface} = \min(t_{mesh}, t_{iso})$. If a surface hit exists, update the ray's maximum interval: $t_{max} \leftarrow t_{surface}$. If no surface hit exists, $t_{max}$ remains unchanged (or is limited by the far plane).
5.  **Volume Integration:** Trace the shortened ray through the Volume BVH. Perform ray marching to integrate the volume density and color from $t_{min}$ to $t_{max}$.
    -   Sample the scalar field at each step.
    -   Apply transfer functions to compute color and opacity.
    -   Use front-to-back compositing to accumulate the final color and opacity.
6.  **Ambient Occlusion (Optional):** For better depth cues, perform additional ray samples in an epsilon region around the surface hit point (if a surface was hit) to compute ambient occlusion. This requires two additional rays per pixel for surface and iso-surface geometry.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_pixel(ray_origin, ray_direction, mesh_bvh, iso_surface_bvh, volume_bvh, 
                 volume_space_transform, clipping_planes, iso_value):
    # 1. Initialize ray interval
    t_min = 0.0
    t_max = far_plane_distance
    
    # Apply clipping planes
    for plane in clipping_planes:
        t_min, t_max = clip_ray_with_plane(ray_origin, ray_direction, plane, t_min, t_max)
        if t_min >= t_max:
            return background_color

    # 2. Trace against Mesh BVH
    t_mesh = infinity
    hit_mesh = False
    mesh_hit = mesh_bvh.intersect(ray_origin, ray_direction)
    if mesh_hit is not None:
        t_mesh = mesh_hit.distance
        hit_mesh = True

    # 3. Trace against Iso-surface BVH
    t_iso = infinity
    hit_iso = False
    
    # Transform ray to volume space
    ray_origin_vol = volume_space_transform.inverse_transform(ray_origin)
    ray_direction_vol = volume_space_transform.transform_direction(ray_direction)
    
    iso_hit = iso_surface_bvh.intersect(ray_origin_vol, ray_direction_vol, iso_value)
    if iso_hit is not None:
        t_iso = iso_hit.distance
        hit_iso = True

    # 4. Determine nearest surface hit
    t_surface = infinity
    if hit_mesh and t_mesh < t_surface:
        t_surface = t_mesh
    if hit_iso and t_iso < t_surface:
        t_surface = t_iso

    # Shorten ray if surface hit found
    if t_surface < infinity:
        t_max = min(t_max, t_surface)

    # 5. Volume Integration
    color = (0.0, 0.0, 0.0, 0.0)
    alpha = 0.0
    
    # Ray marching from t_min to t_max
    step_size = calculate_step_size(volume_bvh, t_max - t_min)
    t = t_min
    while t < t_max:
        # Sample volume data
        sample_pos = ray_origin + ray_direction * t
        scalar_value = volume_bvh.sample(sample_pos)
        
        # Compute color and opacity from transfer function
        c, a = transfer_function(scalar_value)
        
        # Front-to-back compositing
        weight = a * (1.0 - alpha)
        color += c * weight
        alpha += weight
        
        if alpha >= 1.0:
            break
            
        t += step_size

    # 6. Ambient Occlusion (Optional)
    if hit_mesh or hit_iso:
        ao = compute_ambient_occlusion(ray_origin, ray_direction, t_surface)
        color = color * ao

    return color
```

**Parameters:**
-   **Iso-value:** Threshold for iso-surface extraction. Changing this requires rebuilding the Iso-surface BVH.
-   **Clipping Planes:** Defined by plane equations. Implemented by adjusting $t_{min}$ and $t_{max}$.
-   **Ambient Occlusion:** Enabled for depth cues. Uses two additional rays per pixel for surface and iso-surface geometry.

**Edge Cases and Pitfalls:**
-   **Precision:** When transforming rays between world and volume space, ensure consistent precision to avoid gaps or overlaps at boundaries.
-   **Iso-surface Rebuild:** The Iso-surface BVH must be rebuilt whenever the iso-value changes, as the geometry is implicit and dependent on the threshold.
-   **Empty Space Skipping:** Iso-surface BVHs are generally sparser than volume BVHs, allowing for more efficient empty space skipping.

## Operating characteristics
-   **Time Complexity:** Dominated by ray-triangle intersection (hardware-accelerated) and volume ray marching. The number of volume samples is reduced by clipping against surfaces.
-   **Memory Complexity:** Requires storing BVHs for volume, iso-surfaces, and meshes. Iso-surface BVHs are rebuilt on iso-value changes, incurring temporary memory overhead.
-   **Strengths:** Correctly handles occlusion by surfaces; leverages hardware acceleration for meshes; supports complex scenes with mixed geometry.
-   **Limitations:** Requires maintaining multiple BVHs; iso-surface BVH rebuilds can be costly if iso-values change frequently; clipping planes must be carefully managed to avoid artifacts.
-   **When to Use:** Scenes with both explicit surfaces (meshes) and internal volume structures (e.g., medical imaging with skin surface and organs).
-   **When Not to Use:** Pure volume data without surfaces (use pure DVR); pure mesh rendering (use standard ray tracing).

## Background
This method is part of the ExaBricks rendering framework, which supports various rendering modes for structured AMR data. It was developed to evaluate data structures under realistic conditions, including implicit isosurfaces and direct volume rendering. The approach leverages RTX hardware acceleration for mesh intersection and maintains separate BVHs for volume and iso-surfaces to optimize empty space skipping.

## Sources
[S1] Ray Tracing Structured AMR Data Using ExaBricks, https://www.sci.utah.edu/~wald/Publications/2020/exabrick/exabrick.pdf, fetched 2026-10-30
