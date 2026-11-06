# Multi-Mode Rendering with Iso-surfaces and Meshes

Also called: Hybrid Ray Tracing for Iso-surfaces and Meshes, Surface-Volume Hybrid Rendering

## Overview
This method renders scenes containing volumetric data, implicit isosurfaces, and explicit triangle meshes in a single pass by combining hardware-accelerated ray-triangle intersection with volume ray marching [S1]. It solves the problem of efficiently integrating volume density up to a potentially occluding surface (mesh or isosurface) by using separate Bounding Volume Hierarchies (BVHs) for each data type [S1]. Unlike pure volume rendering, it shortens the integration path at the first detected surface hit, and unlike pure mesh rendering, it retains volumetric opacity and color accumulation for the space between the camera and that surface [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A camera ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$, a triangle mesh BVH, an isosurface BVH (built from a scalar field $f(\mathbf{p})$ and isovalue $\tau$), a volume BVH (built from scalar fields), and clipping plane parameters defining $[t_{min}, t_{max}]$.
*   **Outputs:** Final pixel color and opacity.

**Data Structures:**
*   **Mesh BVH:** Standard BVH containing triangle primitives, optimized for hardware-accelerated intersection (e.g., RTX) [S1].
*   **Iso-surface BVH:** BVH representing the implicit surface defined by $f(\mathbf{p}) = \tau$. This structure is rebuilt whenever the isovalue $\tau$ changes [S1].
*   **Volume BVH:** BVH representing the sparse volumetric data, allowing empty space skipping [S1].

**Algorithm Steps:**
1.  **Clip Ray:** Adjust the ray's valid interval to $[t_{min}, t_{max}]$ based on user-defined clipping planes [S1].
2.  **Mesh Intersection:** Trace the ray against the Mesh BVH using hardware-accelerated ray-triangle intersection to find the nearest hit point $t_{mesh}$ [S1].
3.  **Iso-surface Intersection:** Transform the ray into the volume coordinate space. Trace the transformed ray against the Iso-surface BVH to find the nearest hit point $t_{iso}$ [S1].
4.  **Determine Shortest Hit:** Identify the nearest surface hit $t_{hit} = \min(t_{mesh}, t_{iso})$ within the clipped interval. If no surface is hit, $t_{hit}$ remains undefined or extends to $t_{max}$ [S1].
5.  **Shorten Ray:** If a surface hit exists ($t_{hit}$ is defined), the integration range for the volume is shortened to $[t_{min}, t_{hit}]$ [S1].
6.  **Volume Integration:** Trace the (potentially shortened) ray through the Volume BVH. Perform ray marching to integrate the volume density and color up to $t_{hit}$ (or $t_{max}$ if no surface was hit) [S1].
7.  **Shading:** Compute local shading (ambient, diffuse, specular) at the surface hit point if a surface was encountered. Compute ambient occlusion using additional rays if enabled [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_multi_mode_ray(ray_origin, ray_direction, t_min, t_max, 
                          mesh_bvh, iso_bvh, volume_bvh, 
                          volume_space_transform, isovalue, 
                          enable_ao):
    """
    Renders a single ray using hybrid mesh, isosurface, and volume data.
    """
    # Step 1: Clip ray interval
    t_start = t_min
    t_end = t_max
    
    # Step 2: Trace against Mesh BVH
    hit_mesh = False
    t_mesh = infinity
    if mesh_bvh.intersects(ray_origin, ray_direction, t_start, t_end):
        hit_mesh = True
        t_mesh = mesh_bvh.get_nearest_hit_t(ray_origin, ray_direction, t_start, t_end)
        
    # Step 3: Trace against Iso-surface BVH
    # Transform ray to volume space for iso-surface check
    vol_ray_origin, vol_ray_direction = volume_space_transform(ray_origin, ray_direction)
    
    hit_iso = False
    t_iso = infinity
    # Note: The BVH intersection returns t in volume space parameterization
    if iso_bvh.intersects(vol_ray_origin, vol_ray_direction, t_start, t_end):
        hit_iso = True
        t_iso = iso_bvh.get_nearest_hit_t(vol_ray_origin, vol_ray_direction, t_start, t_end)
        
    # Step 4: Determine nearest surface hit
    t_surface = infinity
    if hit_mesh and hit_iso:
        t_surface = min(t_mesh, t_iso)
    elif hit_mesh:
        t_surface = t_mesh
    elif hit_iso:
        t_surface = t_iso
        
    # Step 5: Shorten ray for volume integration
    t_volume_end = t_end
    if t_surface < infinity:
        t_volume_end = t_surface
        
    # Step 6: Volume Integration
    color = vec3(0.0)
    opacity = 0.0
    
    # March from t_start to t_volume_end using Volume BVH for acceleration
    current_t = t_start
    while current_t < t_volume_end:
        # Use BVH to skip empty space and find next sample point
        next_t = volume_bvh.get_next_sample_t(current_t, t_volume_end)
        
        # Sample volume data at current_t
        # Note: Actual sampling requires mapping ray t to voxel coordinates
        sample_pos = ray_origin + current_t * ray_direction
        sample_color, sample_opacity = volume_bvh.sample_volume(sample_pos)
        
        # Alpha compositing
        alpha = sample_opacity * (1.0 - opacity)
        color += sample_color * alpha
        opacity += alpha
        
        if opacity >= 1.0:
            break
            
        current_t = next_t
        
    # Step 7: Surface Shading (if hit)
    if t_surface < infinity:
        hit_pos = ray_origin + t_surface * ray_direction
        
        # Determine if hit was mesh or iso-surface for normal calculation
        normal = vec3(0.0)
        if hit_mesh and t_mesh <= t_iso:
            normal = mesh_bvh.get_normal_at_hit(ray_origin, ray_direction, t_mesh)
        else:
            # For iso-surface, normal is gradient of f at hit_pos
            normal = compute_gradient(isovalue, hit_pos)
            
        # Local shading model
        surface_color = compute_phong_shading(hit_pos, normal, ray_direction)
        
        # Ambient Occlusion (optional, adds 2 rays per pixel)
        ao_factor = 1.0
        if enable_ao:
            ao_factor = compute_ambient_occlusion(hit_pos, normal)
            
        surface_color *= ao_factor
        
        # Composite surface color over volume
        # The surface is opaque, so it replaces the accumulated volume behind it
        # However, standard hybrid rendering usually implies the surface 
        # is the *end* of the volume integration. 
        # If the surface is opaque, the final color is the surface color.
        # If the surface is semi-transparent, standard compositing applies.
        # Based on "shorten the ray... to integrate... up to that point", 
        # the surface is treated as the terminator.
        final_color = surface_color
    else:
        final_color = color
        
    return final_color, opacity

def compute_gradient(isovalue, point):
    """
    Computes the normal for an isosurface using analytic gradients 
    as preferred for quality and speed [S1].
    """
    # Analytic gradient is used by default [S1]
    # This requires access to the underlying scalar field function f(p)
    # grad_f = df/dx, df/dy, df/dz
    return normalize(grad_f(point))
```

**Parameters:**
*   **Isovalue ($\tau$):** The scalar threshold for the isosurface. Changing this requires rebuilding the Iso-surface BVH [S1].
*   **Clipping Planes:** Defined by $t_{min}$ and $t_{max}$ on the ray [S1].
*   **Ambient Occlusion:** Enabled by default for depth cues, requiring two additional rays per pixel for surface and iso-surface geometry [S1].

**Edge Cases and Pitfalls:**
*   **Iso-surface BVH Rebuild:** The Iso-surface BVH must be rebuilt whenever the isovalue changes, as the geometry is implicit and sparse [S1].
*   **Gradient Continuity:** Analytic gradients are preferred over central differences for speed and quality, though they may not always be continuous [S1].
*   **Multifield Support:** The system supports color-mapping an isosurface from one scalar field using colors from another (e.g., flow velocity on vorticity) [S1].

## Operating characteristics
*   **Time Complexity:** Dominated by BVH traversal for all three data structures (Mesh, Iso, Volume). Hardware-accelerated mesh intersection is highly efficient [S1].
*   **Memory Complexity:** Requires maintaining three separate BVHs. The Iso-surface BVH size depends on the sparsity of the isosurface [S1].
*   **Strengths:** Efficient empty space skipping via BVHs; supports hybrid scenes with meshes, implicit surfaces, and volumes; hardware acceleration for meshes [S1].
*   **Limitations:** Iso-surface BVH must be rebuilt on isovalue change; requires significant memory for multiple BVHs [S1].

## Background
This method is part of the ExaBricks framework for ray tracing structured AMR data, developed by Wald et al. [S1]. It leverages the ExaBricks data structure to handle sparse volumetric data and integrates with modern GPU ray-tracing hardware (RTX) for mesh intersection [S1].

## Sources
[S1] Ray Tracing Structured AMR Data Using ExaBricks, https://www.sci.utah.edu/~wald/Publications/2020/exabrick/exabrick.pdf, fetched 2026-10-30
