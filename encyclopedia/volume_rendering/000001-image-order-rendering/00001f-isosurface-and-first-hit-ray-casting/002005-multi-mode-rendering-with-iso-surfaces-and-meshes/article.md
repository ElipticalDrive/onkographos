# Multi-Mode Rendering with Iso-surfaces and Meshes

Also called: Hybrid Ray Tracing for Iso-surfaces and Meshes, Surface-Volume Hybrid Rendering

## Overview
This method renders scenes containing volumetric data, implicit isosurfaces, and explicit triangle meshes in a single pass by combining hardware-accelerated ray-triangle intersection with volume ray marching [S1]. It solves the problem of efficiently integrating volume density up to a potentially occluding surface by using separate Bounding Volume Hierarchies (BVHs) for each data type [S1]. Unlike pure volume rendering, it shortens the integration path at the first detected surface hit, and unlike pure mesh rendering, it retains volumetric opacity and color accumulation for the space between the camera and that surface [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A camera ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$, a triangle mesh BVH, an isosurface BVH (built from a scalar field $f(\mathbf{p})$ and isovalue $\tau$), a volume BVH (built from scalar fields), and clipping plane parameters defining $[t_{\text{min}}, t_{\text{max}}]$ [S1].
*   **Outputs:** Final pixel color and opacity [S1].

**Data Structures:**
*   **Mesh BVH:** Standard BVH containing triangle primitives, optimized for hardware-accelerated intersection [S1].
*   **Iso-surface BVH:** BVH representing the implicit surface defined by $f(\mathbf{p}) = \tau$ [S1]. This structure is rebuilt whenever the isovalue $\tau$ changes [S1].
*   **Volume BVH:** BVH representing the sparse volumetric data, allowing empty space skipping [S1].

**Algorithm Steps:**
1.  **Clip Ray:** Adjust the ray's valid interval to $[t_{\text{min}}, t_{\text{max}}]$ based on user-defined clipping planes [S1].
2.  **Mesh Intersection:** Trace the ray against the Mesh BVH using hardware-accelerated ray-triangle intersection to find the nearest hit point $t_{\text{mesh}}$ [S1].
3.  **Iso-surface Intersection:** Transform the ray into the volume coordinate space [S1]. Trace the transformed ray against the Iso-surface BVH to find the nearest hit point $t_{\text{iso}}$ [S1].
4.  **Determine Shortest Hit:** Identify the nearest surface hit $t_{\text{hit}} = \min(t_{\text{mesh}}, t_{\text{iso}})$ within the clipped interval [S1]. If no surface is hit, $t_{\text{hit}}$ is set to $t_{\text{max}}$ [S1].
5.  **Shorten Ray:** If a surface hit exists, the integration range for the volume is shortened to $[t_{\text{min}}, t_{\text{hit}}]$ [S1].
6.  **Volume Integration:** Trace the shortened ray through the Volume BVH to integrate the volume density and color up to $t_{\text{hit}}$ [S1].
7.  **Shading:** Compute local shading at the surface hit point if a surface was encountered [S1]. Compute ambient occlusion using two rays per pixel for surface and iso-surface geometry [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_multi_mode_ray(ray_origin, ray_direction, t_min, t_max, 
                          mesh_bvh, iso_bvh, volume_bvh, 
                          volume_space_transform, scalar_field, isovalue,
                          enable_ao):
    # Step 1: Clip ray interval
    t_start = t_min
    t_end = t_max
    
    # Step 2: Trace against Mesh BVH
    hit_mesh = False
    t_mesh = float('inf')
    if mesh_bvh.intersects(ray_origin, ray_direction, t_start, t_end):
        hit_mesh = True
        t_mesh = mesh_bvh.get_nearest_hit_t(ray_origin, ray_direction, t_start, t_end)
        
    # Step 3: Trace against Iso-surface BVH
    vol_origin, vol_direction = volume_space_transform(ray_origin, ray_direction)
    hit_iso = False
    t_iso = float('inf')
    if iso_bvh.intersects(vol_origin, vol_direction, t_start, t_end):
        hit_iso = True
        t_iso = iso_bvh.get_nearest_hit_t(vol_origin, vol_direction, t_start, t_end)
        
    # Step 4: Determine nearest surface hit
    t_surface = float('inf')
    if hit_mesh and hit_iso:
        t_surface = min(t_mesh, t_iso)
    elif hit_mesh:
        t_surface = t_mesh
    elif hit_iso:
        t_surface = t_iso
        
    # Step 5: Shorten ray for volume integration
    t_volume_end = t_surface
        
    # Step 6: Volume Integration
    color = vec3(0.0)
    opacity = 0.0
    current_t = t_start
    while current_t < t_volume_end:
        next_t = volume_bvh.get_next_sample_t(current_t, t_volume_end)
        sample_pos = ray_origin + current_t * ray_direction
        sample_color, sample_opacity = volume_bvh.sample_volume(sample_pos)
        alpha = sample_opacity * (1.0 - opacity)
        color += sample_color * alpha
        opacity += alpha
        if opacity >= 1.0:
            break
        current_t = next_t
        
    # Step 7: Surface Shading
    if t_surface < float('inf'):
        hit_pos = ray_origin + t_surface * ray_direction
        normal = vec3(0.0)
        if hit_mesh and t_mesh <= t_iso:
            normal = mesh_bvh.get_normal_at_hit(ray_origin, ray_direction, t_mesh)
        else:
            normal = scalar_field.gradient(hit_pos)
            
        # Standard Phong lighting model [G]
        diffuse = max(dot(normal, light_dir), 0.0)
        view_dir = normalize(-ray_direction)
        reflect_dir = reflect(-light_dir, normal)
        specular = pow(max(dot(view_dir, reflect_dir), 0.0), 32.0)
        surface_color = ambient_color + diffuse * diffuse_color + specular * specular_color
        
        # Ambient occlusion uses two rays per pixel for depth cues [S1]
        ao_factor = 1.0
        if enable_ao:
            ray1 = trace_ray(hit_pos + normal * epsilon, normal, t_max)
            ray2 = trace_ray(hit_pos - normal * epsilon, -normal, t_max)
            ao_factor = 0.5 * (1.0 if ray1.hit else 0.0) + 0.5 * (1.0 if ray2.hit else 0.0)
            
        surface_color *= ao_factor
        final_color = surface_color
    else:
        final_color = color
        
    return final_color, opacity
```

**Parameters:**
*   **Isovalue ($\tau$):** The scalar threshold for the isosurface [S1]. Changing this requires rebuilding the Iso-surface BVH [S1].
*   **Clipping Planes:** Defined by $t_{\text{min}}$ and $t_{\text{max}}$ on the ray [S1].
*   **Ambient Occlusion:** Supported for better depth cues, using two rays per pixel on surface and iso-surface geometry [S1].

**Edge Cases and Pitfalls:**
*   **Iso-surface BVH Rebuild:** The Iso-surface BVH must be rebuilt whenever the isovalue changes, as the geometry is implicit and sparse [S1].
*   **Gradient Continuity:** Analytic gradients are used by default, though they may not always be continuous [S1].
*   **Multifield Support:** The system supports color-mapping an isosurface computed on one scalar field with colors computed from another [S1].

## Operating characteristics
*   **Time Complexity:** Dominated by BVH traversal for all three data structures (Mesh, Iso, Volume) [S1].
*   **Memory Complexity:** Requires maintaining three separate BVHs [S1].
*   **Strengths:** Efficient empty space skipping via BVHs; supports hybrid scenes with meshes, implicit surfaces, and volumes; hardware acceleration for meshes [S1].
*   **Limitations:** Iso-surface BVH must be rebuilt on isovalue change; requires significant memory for multiple BVHs [S1].

## Background
This method is described in the paper "Ray Tracing Structured AMR Data Using ExaBricks" [S1]. Not given in the sources.

## Sources
[S1] Ray Tracing Structured AMR Data Using ExaBricks, https://www.sci.utah.edu/~wald/Publications/2020/exabrick/exabrick.pdf, fetched 2026-10-30
