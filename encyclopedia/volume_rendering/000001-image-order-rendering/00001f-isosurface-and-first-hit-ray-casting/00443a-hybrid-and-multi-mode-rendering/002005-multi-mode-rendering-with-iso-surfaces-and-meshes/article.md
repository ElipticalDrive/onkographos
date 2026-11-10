# Multi-Mode Rendering with Iso-surfaces and Meshes

Also called: Hybrid Ray Tracing for Iso-surfaces and Meshes, Surface-Volume Hybrid Rendering

## Overview
Multi-Mode Rendering with Iso-surfaces and Meshes is an image-order rendering technique that combines hardware-accelerated ray-triangle intersection for explicit surface geometry (meshes) and implicit isosurfaces with volume ray marching for volumetric data [S1]. It solves the problem of efficiently rendering scenes containing both discrete volumetric data and sharp surface boundaries by using clipping planes to restrict volume integration to the region in front of the nearest surface [S1]. Unlike pure volume rendering, which integrates through the entire ray, or pure mesh rendering, which lacks internal density, this method shortens the ray at the first surface hit to prevent occlusion of volume data behind the surface [S1]. It is distinguished from [Hybrid Isosurface/DVR Rendering](8367-hybrid-isosurface-dvr-rendering.md) by its explicit support for hardware-accelerated triangle meshes alongside implicit surfaces, and from [Hybrid Implicit Isosurface Ray Tracing](11176-hybrid-implicit-isosurface-ray-tracing.md) by its specific traversal order prioritizing mesh BVHs before iso-surface BVHs.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of triangle meshes, a scalar field defining an isosurface, a scalar field defining volume density, camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$), and rendering mode flags (volume, iso-surface, mesh, ambient occlusion).
*   **Outputs:** A pixel color value.

**Data Structures:**
*   **Mesh BVH:** A bounding volume hierarchy for triangle meshes, optimized for hardware-accelerated intersection (e.g., RTX) [S1].
*   **Iso-surface BVH:** A bounding volume hierarchy for the implicit isosurface. It is rebuilt whenever the iso-value threshold changes [S1].
*   **Volume BVH:** A bounding volume hierarchy for the volumetric data, used for empty space skipping during ray marching [S1].
*   **Clipping Planes:** Defined by modifying the ray's $[t_{min}, t_{max}]$ interval [S1].

**Steps:**
1.  **Ray Definition:** Define the primary ray $r(t) = \mathbf{o} + t\mathbf{d}$ for the current pixel [G].
2.  **Mesh Intersection:** Trace the ray against the Mesh BVH using hardware-accelerated ray-triangle intersection to find the nearest intersection distance $t_{mesh}$ [S1].
3.  **Iso-surface Intersection:** Transform the ray into the volume space. Trace the transformed ray against the Iso-surface BVH to find the nearest intersection distance $t_{iso}$ [S1].
4.  **Ray Shortening:** Determine the effective end of the volume integration range $t_{end}$.
    *   If a surface hit exists ($t_{mesh} < \infty$ or $t_{iso} < \infty$), set $t_{end} = \min(t_{mesh}, t_{iso})$ [S1].
    *   If no surface hit exists, $t_{end}$ is determined by the volume domain boundary or clipping planes [S1].
    *   Apply clipping planes by clamping $t_{end}$ to the specified $[t_{min}, t_{max}]$ interval [S1].
5.  **Volume Integration:** Trace the shortened ray from $t_{start}$ (typically 0 or $t_{min}$) to $t_{end}$ through the Volume BVH.
    *   Perform ray marching, skipping empty regions defined by the Volume BVH [S1].
    *   Sample the scalar field and apply transfer functions to compute color and opacity.
    *   Composite the volume samples using standard volume rendering integration (e.g., front-to-back compositing) [S1].
6.  **Surface Shading (Optional):** If the ray hits a surface ($t_{hit} = t_{end}$), compute the surface color using local lighting models (ambient, diffuse, specular) based on the surface normal [S3, unofficial][S1]. The normal is computed from the gradient $\nabla f$ for isosurfaces or from the triangle geometry for meshes [S3, unofficial][S1].
7.  **Ambient Occlusion (Optional):** For better depth cues, trace two additional rays per pixel (one for surface, one for iso-surface geometry) to compute ambient occlusion factors [S1].
8.  **Final Color:** Combine the volume color (from step 5) and surface color (from step 6) if applicable. If a surface is hit, the volume color is typically occluded by the surface color, or the surface is rendered on top of the integrated volume up to $t_{end}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_pixel(ray_o, ray_d, mesh_bvh, iso_bvh, vol_bvh, iso_value, clipping_tmin, clipping_tmax):
    # Step 2: Mesh Intersection
    t_mesh = mesh_bvh.intersect(ray_o, ray_d)
    
    # Step 3: Iso-surface Intersection
    # Transform ray to volume space
    ray_o_vol, ray_d_vol = transform_to_volume_space(ray_o, ray_d)
    t_iso = iso_bvh.intersect(ray_o_vol, ray_d_vol)
    
    # Step 4: Ray Shortening
    t_surface = min(t_mesh, t_iso) if (t_mesh != INF and t_iso != INF) else INF
    
    # Apply clipping planes
    t_end = min(t_surface, clipping_tmax)
    t_start = max(0.0, clipping_tmin)
    
    # Ensure valid range
    if t_start >= t_end:
        return background_color
        
    # Step 5: Volume Integration
    vol_color, vol_opacity = 0.0, 0.0
    t = t_start
    while t < t_end:
        # Skip empty space using Volume BVH
        t = vol_bvh.skip_empty(ray_o, ray_d, t, t_end)
        
        if t >= t_end:
            break
            
        # Sample volume
        density = sample_volume(ray_o, ray_d, t)
        color, opacity = apply_transfer_function(density)
        
        # Composite
        alpha = opacity * (1.0 - vol_opacity)
        vol_color += color * alpha
        vol_opacity += alpha
        
        t += step_size # Adaptive step size based on BVH leaf size
        
    # Step 6: Surface Shading
    surface_color = 0.0
    if t_surface < t_end:
        hit_point = ray_o + t_surface * ray_d
        normal = compute_normal(hit_point, mesh_bvh, iso_bvh)
        surface_color = shade(hit_point, normal, ray_d)
        
    # Step 8: Final Color
    # If surface is hit, it occludes the volume behind it.
    # The volume is integrated only up to t_surface.
    if t_surface < t_end:
        return surface_color
    else:
        return vol_color
```

**Parameters:**
*   **Iso-value:** Threshold for isosurface extraction. Changing this requires rebuilding the Iso-surface BVH [S1].
*   **Clipping Planes:** Defined by $[t_{min}, t_{max}]$ intervals on the ray [S1].
*   **Ambient Occlusion:** Enabled via additional ray traces for depth cues [S1].

**Edge Cases and Pitfalls:**
*   **Iso-value Change:** The Iso-surface BVH must be rebuilt when the iso-value changes, as the geometry is implicit [S1].
*   **Precision:** Analytic gradients are used by default for isosurface normals as they provide superior quality and speed compared to central differences, though they may not be continuous [S1].
*   **Multiple Fields:** The system supports color-mapping an isosurface computed on one scalar field with colors from another, requiring careful handling of field references in bricks/regions [S1].

## Subtopics

### Hybrid Isosurface/DVR Rendering
Combines isosurface ray casting with direct volume rendering, but does not explicitly support hardware-accelerated triangle meshes in the same traversal pipeline [S1].

### Hybrid Implicit Isosurface Ray Tracing
Focuses on implicit surfaces and may not include the explicit mesh BVH traversal or the specific clipping plane implementation described here [S1].

### Mesh-based Neural Rendering
Uses neural networks for rendering meshes, differing from the traditional ray-triangle intersection and volume marching approach [S1].

## Operating characteristics
*   **Time Complexity:** Dominated by ray traversal in BVHs. Mesh BVH intersection is hardware-accelerated, providing fast $O(\log n)$ performance [S1]. Iso-surface BVH intersection is also $O(\log n)$ but requires rebuilding when the iso-value changes [S1]. Volume ray marching is $O(m)$ where $m$ is the number of samples, but empty space skipping via the Volume BVH reduces this significantly [S1].
*   **Memory Complexity:** Requires storing three BVHs: Mesh, Iso-surface, and Volume. The Iso-surface BVH must be rebuilt when the iso-value changes, incurring temporary memory and time costs [S1].
*   **Performance:** Hardware-accelerated ray-triangle intersection (RTX) provides significant speedups for mesh rendering [S1]. The method is efficient for scenes with sparse isosurfaces due to empty space skipping [S1].
*   **Strengths:** Supports multiple rendering modes (volume, iso-surface, mesh) in a single pipeline. Hardware acceleration for meshes improves performance. Clipping planes allow for interactive exploration.
*   **Limitations:** Iso-surface BVH rebuilds can be costly if the iso-value changes frequently. Analytic gradients may not be continuous, potentially affecting image quality in some cases [S1].
*   **When to Use:** Scenes containing both volumetric data and sharp surface boundaries (meshes or isosurfaces) where occlusion of volume by surfaces is important.
*   **When Not to Use:** Scenes with only volumetric data (use pure DVR) or only meshes (use pure rasterization/ray tracing).

## Background
The method is described in the context of the ExaBricks renderer, which targets structured AMR (Adaptive Mesh Refinement) data [S1]. It was developed to support a variety of rendering modes for evaluating data structures under realistic conditions, including implicit isosurfaces and direct volume rendering [S1]. The approach leverages hardware-accelerated ray-triangle intersection (RTX) for meshes and BVHs for empty space skipping in volumes and isosurfaces [S1].

## Sources
[S1] Ray Tracing Structured AMR Data Using ExaBricks, https://www.sci.utah.edu/~wald/Publications/2020/exabrick/exabrick.pdf, fetched 2026-10-30
