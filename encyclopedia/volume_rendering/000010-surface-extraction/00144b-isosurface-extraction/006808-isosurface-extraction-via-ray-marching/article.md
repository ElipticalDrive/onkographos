# Isosurface Extraction via Ray Marching

Also called: First-hit ray casting with isosurface thresholding, Marching Cubes-like Ray Intersection

## Overview
Isosurface extraction via ray marching is a direct volume rendering technique that identifies and renders the boundary of a scalar field by detecting where a viewing ray crosses a specific isovalue threshold [S1, S2, unofficial]. Unlike grid-based meshing methods such as [Marching Cubes](30-isosurface-extraction.md) which generate an explicit polygonal mesh, this method computes surface geometry and shading per-pixel during the ray traversal, allowing for interactive isovalue updates without re-meshing [S2, unofficial]. It is distinguished from [Topologically Accurate Dual Isosurfacing Using Ray Intersection](800-topologically-accurate-dual-isosurfacing-using-ray-intersect.md) by its reliance on simple threshold crossing and finite-difference normals rather than topological guarantees or dual-contouring optimization [S1, S2, unofficial].

## How it works
**Inputs:**
*   Scalar volume data $V$ defined on a regular grid of dimensions $D_x \times D_y \times D_z$.
*   Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ for each pixel.
*   Isosurface threshold value $T$ (isovalue).
*   Step size $s$ for coarse marching (optional, often fixed or adaptive).
*   Normal smoothing parameter $u_{IsoSmooth}$ [S1, unofficial].
*   Resolution $R$ of the volume grid [S1, unofficial].

**Outputs:**
*   Final pixel color $C_{final}$ and opacity $\alpha_{final}$.

**Data Structures:**
*   Volume $V$: A 3D array or texture where $V(x,y,z)$ returns the scalar density at grid coordinates.
*   State variables per ray: `inside` (boolean, initially false), `density` (scalar), `pos` (3D vector), `normal` (3D vector).

**Algorithm Steps:**

1.  **Ray Marching and Crossing Detection:**
    Iterate along the ray from the near plane to the far plane with a fixed step size or adaptive steps. For each sample point $\mathbf{p}$:
    *   Sample density: $d = V(\mathbf{p})$.
    *   Check for isosurface crossing:
        *   If `inside` is false and $d \ge T$, a crossing is detected. Set `inside` = true.
        *   If `inside` is true and $d < T$, the ray has exited the surface. Set `inside` = false.
    *   If a crossing is detected (transitioning from false to true), proceed to refinement. If the ray exits (true to false), the surface is behind the current segment; depending on implementation, one might refine the exit point or stop if only the front surface is desired [S1, S2, unofficial].

2.  **Intersection Refinement:**
    Refine the exact intersection position using binary search over the segment where the crossing occurred [S1, unofficial].
    *   Let $a$ be the distance along the ray at the last sample *before* the crossing (where density was below threshold).
    *   Let $b$ be the distance along the ray at the current sample (where density is above threshold).
    *   Perform binary search for $N=5$ iterations:
        1.  $mid = (a + b) / 2$.
        2.  $\mathbf{p}_{mid} = \mathbf{o} + \mathbf{d} \cdot mid$.
        3.  $d_{mid} = V(\mathbf{p}_{mid})$.
        4.  If $d_{mid} < T$, set $a = mid$. Else, set $b = mid$.
    *   The refined position is $\mathbf{p}_{hit} = \mathbf{o} + \mathbf{d} \cdot \frac{a+b}{2}$.

3.  **Normal Computation:**
    Compute the surface normal $\mathbf{n}$ using finite differences [S1, unofficial].
    *   Define shift vector magnitude $\Delta = \frac{u_{IsoSmooth}}{R}$ [S1, unofficial].
    *   Sample densities at neighbors:
        *   $d_x = V(\mathbf{p}_{hit} + (\Delta, 0, 0))$
        *   $d_y = V(\mathbf{p}_{hit} + (0, \Delta, 0))$
        *   $d_z = V(\mathbf{p}_{hit} + (0, 0, \Delta))$
    *   Compute gradient (unnormalized normal):
        $$ \nabla V \approx \begin{bmatrix} d_x - V(\mathbf{p}_{hit}) \\ d_y - V(\mathbf{p}_{hit}) \\ d_z - V(\mathbf{p}_{hit}) \end{bmatrix} $$
    *   Normalize: $\mathbf{n}_{local} = \frac{\nabla V}{\|\nabla V\|}$.
    *   Transform to world/view space if necessary using normal matrix $M_N$: $\mathbf{n} = M_N \cdot \mathbf{n}_{local}$ [S1, unofficial].
    *   *Alternative:* If parameter $u_{IsoWalls} > 0$, return fixed normals based on the bounding box limits instead of gradient-based normals [S1, unofficial].

4.  **Shading and Compositing:**
    *   Compute surface color $C_{surf}$ using lighting models (e.g., Phong) with $\mathbf{n}$ [S1, unofficial].
    *   Composite the surface color into the accumulated ray color using front-to-back blending with the surface opacity $\alpha_{surf}$ [S1, unofficial]:
        $$ C_{final} = C_{surf} \cdot \alpha_{surf} + C_{accumulated} \cdot (1 - \alpha_{surf}) $$
    *   Update accumulated opacity: $\alpha_{accumulated} = \alpha_{accumulated} + \alpha_{surf} \cdot (1 - \alpha_{accumulated})$.
    *   If $\alpha_{accumulated} \ge 1$, terminate ray marching [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march_isosurface(ray_origin, ray_direction, volume, isovalue, resolution, iso_smooth, iso_walls, normal_matrix):
    """
    Renders an isosurface by ray marching.
    :param ray_origin: vec3
    :param ray_direction: vec3
    :param volume: 3D array or texture lookup function
    :param isovalue: float
    :param resolution: float (grid dimension size)
    :param iso_smooth: float (smoothing factor for normal calculation)
    :param iso_walls: float (threshold to enable wall normals)
    :param normal_matrix: mat3 (normal transformation matrix)
    :return: (color, opacity)
    """
    accumulated_color = vec3(0.0)
    accumulated_opacity = 0.0
    inside = False
    step_size = 0.1  # Example fixed step size; could be adaptive
    
    # Determine ray length based on bounding box intersection
    ray_length = compute_ray_length(ray_origin, ray_direction)
    
    pos = ray_origin
    for t in range(0, int(ray_length / step_size)):
        density = volume.sample(pos)
        
        # Check for crossing
        if not inside and density >= isovalue:
            inside = True
            # Refine intersection
            a = t * step_size
            b = (t + 1) * step_size
            
            # Binary search refinement (5 iterations)
            for _ in range(5):
                mid = (a + b) * 0.5
                mid_pos = ray_origin + ray_direction * mid
                mid_density = volume.sample(mid_pos)
                
                if mid_density < isovalue:
                    a = mid
                else:
                    b = mid
            
            hit_pos = ray_origin + ray_direction * ((a + b) * 0.5)
            
            # Compute Normal
            delta = iso_smooth / resolution
            shift_x = vec3(delta, 0.0, 0.0)
            shift_y = vec3(0.0, delta, 0.0)
            shift_z = vec3(0.0, 0.0, delta)
            
            if iso_walls > 0.0:
                # Return fixed wall normal (simplified example)
                normal_local = vec3(0.0, 0.0, 1.0) 
            else:
                grad_x = volume.sample(hit_pos + shift_x) - density
                grad_y = volume.sample(hit_pos + shift_y) - density
                grad_z = volume.sample(hit_pos + shift_z) - density
                normal_local = normalize(vec3(grad_x, grad_y, grad_z))
            
            # Transform normal
            normal = normal_matrix * normal_local
            
            # Shading
            surface_color = compute_lighting(hit_pos, normal, ray_direction)
            surface_opacity = 1.0 # Or defined by transfer function
            
            # Front-to-back blending
            accumulated_color = surface_color * surface_opacity + accumulated_color * (1.0 - surface_opacity)
            accumulated_opacity = accumulated_opacity + surface_opacity * (1.0 - accumulated_opacity)
            
            if accumulated_opacity >= 1.0:
                break
                
        elif inside and density < isovalue:
            inside = False
            # Optional: Refine exit point if back-face rendering is required
            # For standard front-facing isosurface, we may stop or continue
            pass
            
        pos = ray_origin + ray_direction * ((t + 1) * step_size)
        
    return accumulated_color, accumulated_opacity
```

**Parameters:**
*   `isovalue`: The scalar threshold defining the surface. Can be updated interactively [S2, unofficial].
*   `iso_smooth`: Controls the step size for finite difference normal calculation. Divided by `resolution` [S1, unofficial].
*   `iso_walls`: If $> 0$, enables boundary wall normals (fixed normals at bounding box limits) instead of gradient-based normals [S1, unofficial].

**Pitfalls:**
*   The binary search assumes monotonicity between samples. If the density oscillates rapidly, the intersection might be inaccurate [S1, unofficial].
*   Finite difference normals are sensitive to the step size $\Delta$; too large causes blurring, too small causes noise [S1, unofficial].

## Subtopics

### Interactive Isosurface Picking
A technique to generate an isosurface from a scalar field based on a 3D location pointed to by a stylus, allowing interactive exploration of isovalues without manual entry [S3]. The user points to a specific 3D location within the volume, and the isosurface is computed using the scalar value at that point [S3]. Optionally, a low-resolution preview of the isosurface is displayed in real-time during stylus displacement until the desired value is reached [S3].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot I)$, where $N$ is the number of pixels and $I$ is the number of steps per ray. The binary search adds a constant factor ($5$ iterations).
*   **Memory Complexity:** $O(1)$ per ray, excluding the storage of the volume data itself.
*   **Performance:** Performance depends heavily on the volume size and the density of the isosurface. Empty-space skipping can improve performance [S2, unofficial].
*   **Strengths:** No explicit mesh generation required; allows real-time isovalue adjustment; simple to implement on GPU shaders [S1, S2, unofficial].
*   **Limitations:** No topological guarantees (can produce holes or artifacts if sampling is insufficient); normals are approximate [S1, S2, unofficial].
*   **When to use:** Interactive visualization, real-time exploration, simple surface rendering.
*   **When not to use:** When topological accuracy is critical, or when a mesh is required for further processing (e.g., physics simulation, 3D printing).

## Background
Isosurface extraction via ray marching is a fundamental technique in direct volume rendering. It is often implemented in fragment shaders for real-time visualization. The method leverages the simplicity of ray casting to avoid the complexity of mesh generation algorithms like Marching Cubes [S1, S2, unofficial]. Interactive picking techniques extend this method to allow users to select isovalue thresholds by pointing to structures in the volume [S3].

## Sources
[S1] WebGL-Volume-rendering, https://github.com/jarusified/WebGL-Volume-rendering, fetched 2026-10-27 (unofficial)
[S2] Volume Rendering, https://docslib.org/doc/4315965/volume-rendering, fetched 2026-10-28 (unofficial)
[S3] A portable interface for tangible exploration of volumetric data, https://doi.org/10.1145/2671015.2671130, fetched 2026-10-29
