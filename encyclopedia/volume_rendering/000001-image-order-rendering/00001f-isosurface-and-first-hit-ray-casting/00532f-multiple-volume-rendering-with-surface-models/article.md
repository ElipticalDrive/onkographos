# Multiple Volume Rendering with Surface Models

Also called: Overlaying volume data sets with 3D surface models, Bounded volume compositing

## Overview
This method renders multiple volumetric data sets overlaid on a 3D surface model (e.g., an architectural building) by computing the contribution of each volume set individually and combining them [S1]. It solves the problem of efficiently visualizing internal volume structures within a bounded exterior geometry, optimizing memory access patterns for graphics hardware [S1]. Unlike standard ray casting that treats the volume as an infinite medium, this method distinguishes between "bounded" volumes (which have a geometric boundary like a wall) and "un-bounded" volumes, stopping ray integration at the nearest surface boundary for bounded sets [S1].

## How it works
**Inputs:**
- A 3D surface model (mesh) defining boundaries (e.g., building walls) [S1].
- $M$ volumetric data sets, each with associated transfer functions and shader methods [S1].
- Camera position and viewing direction [S1].
- A pre-computed depth buffer (z-buffer) of the surface model from the current camera perspective [S1].

**Outputs:**
- Final pixel color $\mathbf{C}_{out}$ [S1].

**Data Structures:**
- **Surface Depth Buffer:** A 2D array storing the distance $t_{surface}$ from the camera origin to the nearest surface intersection for each pixel [S1].
- **Volume Data Sets:** Array of $M$ volumes, each containing voxel data and a classification of its shader method type (Bounded or Un-bounded) [S1].
- **Shader Method Classification:** Each volume is assigned one of seven shader methods, categorized into two groups:
    1.  **Penetrating:** Rays fully penetrate the volume (Methods 1, 2, 4, 7) [S1].
    2.  **Terminating:** Rays end at a point inside the volume (Methods 3, 5, 6) [S1].

**Algorithm Steps:**
1.  **Surface Pre-computation:** Render the 3D surface model to a depth buffer to determine the boundary distance $t_{surface}$ for every pixel [S1].
2.  **Ray Initialization:** For each pixel, define the ray $r(t)$ starting at the camera origin [G].
3.  **Volume Classification:** Identify which of the $M$ volume data sets are "bounded" (have a geometric limit defined by the surface model) and which are "un-bounded" [S1].
4.  **Boundary Determination:** For the current ray, find the nearest boundary point $t_{boundary}$ among all "bounded" volume sets. If no bounded volumes exist, $t_{boundary} = \infty$ [S1].
5.  **Un-bounded Volume Integration:** For all "un-bounded" volume sets, perform ray marching from $t_{near}$ to $t_{boundary}$ (or $t_{far}$ if no boundary is closer). Compute the accumulated color and opacity contribution $\mathbf{C}_{unbounded}$ using standard front-to-back compositing [S1].
6.  **Bounded Volume Integration:** For each "bounded" volume set, perform ray marching from $t_{near}$ to $t_{boundary}$ (the specific boundary for that volume). Compute the contribution $\mathbf{C}_{bounded, i}$ for each volume $i$ individually [S1].
7.  **Combination:** Combine the contributions of all volume sets. The specific combination logic depends on the shader method group:
    -   For **Penetrating** methods, contributions are composited through the volume.
    -   For **Terminating** methods, the ray integration stops at the computed interior point or boundary [S1].
    -   The final color is calculated using an optional transfer function applied to the combined data [S1].
8.  **Optimization:** Memory access patterns are optimized by compacting address blocks to maximize memory caching during the individual volume computations [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_multiple_volumes_with_surface(surface_depth_buffer, volumes, camera):
    """
    Render multiple volumes overlaid on a surface model.
    
    Args:
        surface_depth_buffer: 2D array of distances to surface boundaries per pixel.
        volumes: List of Volume objects, each with:
            - data: 3D voxel grid
            - transfer_function: function mapping voxel value to (color, opacity)
            - shader_type: Enum {PENETRATING, TERMINATING}
            - is_bounded: Boolean
        camera: Camera object with origin and direction generation.
        
    Returns:
        image: 2D array of final pixel colors.
    """
    height, width = surface_depth_buffer.shape
    image = np.zeros((height, width, 3))
    
    for y in range(height):
        for x in range(width):
            # 1. Get surface boundary distance for this pixel
            t_surface = surface_depth_buffer[y, x]
            
            # 2. Initialize accumulated color and opacity
            acc_color = np.array([0.0, 0.0, 0.0])
            acc_opacity = 0.0
            
            # 3. Separate volumes into bounded and un-bounded
            bounded_volumes = [v for v in volumes if v.is_bounded]
            unbounded_volumes = [v for v in volumes if not v.is_bounded]
            
            # 4. Determine the nearest boundary among bounded volumes
            # If no bounded volumes, t_boundary is effectively infinity
            t_boundary = np.inf
            if bounded_volumes:
                # For simplicity, assume all bounded volumes share the same 
                # geometric boundary defined by the surface model.
                # In complex scenarios, each volume might have its own boundary mesh.
                t_boundary = t_surface 
            
            # 5. Process Un-bounded Volumes
            # These are integrated up to the nearest boundary (or full depth if no boundary)
            # Note: Standard ray casting logic applies here, but limited by t_boundary
            for vol in unbounded_volumes:
                # Ray march from t_near to min(t_far, t_boundary)
                # Compute contribution using front-to-back compositing
                vol_color, vol_opacity = ray_march_volume(
                    vol, camera, t_near=0.0, t_end=min(t_boundary, vol.max_depth)
                )
                
                # Composite into accumulator
                # C_out = C_acc + (1 - alpha_acc) * C_vol * alpha_vol
                # alpha_out = alpha_acc + (1 - alpha_acc) * alpha_vol
                acc_color = acc_color + (1 - acc_opacity) * vol_color * vol_opacity
                acc_opacity = acc_opacity + (1 - acc_opacity) * vol_opacity

            # 6. Process Bounded Volumes
            # These are integrated up to the specific boundary
            for vol in bounded_volumes:
                # Ray march from t_near to t_boundary
                vol_color, vol_opacity = ray_march_volume(
                    vol, camera, t_near=0.0, t_end=t_boundary
                )
                
                # Composite into accumulator
                acc_color = acc_color + (1 - acc_opacity) * vol_color * vol_opacity
                acc_opacity = acc_opacity + (1 - acc_opacity) * vol_opacity

            # 7. Apply optional transfer function for final color adjustment
            # (Assuming transfer function was applied during ray march, 
            # this step might handle global adjustments or specific shader logic)
            image[y, x] = acc_color

    return image

def ray_march_volume(volume, camera, t_near, t_end):
    """
    Standard front-to-back ray marching for a single volume.
    Optimized for memory access patterns [S1].
    """
    acc_color = np.array([0.0, 0.0, 0.0])
    acc_opacity = 0.0
    
    # Generate sample points along the ray
    # Number of samples N depends on resolution and step size
    N = 100 # Placeholder for implementation detail
    dt = (t_end - t_near) / N
    
    for i in range(N):
        t = t_near + i * dt
        p = camera.origin + t * camera.direction
        
        # Sample volume value
        v = volume.sample(p)
        
        # Map to color and opacity using transfer function
        c, alpha = volume.transfer_function(v)
        
        # Correct opacity for step size (Beer-Lambert)
        # alpha = 1 - exp(-sigma * dt)
        # [G] Assuming sigma is derived from v or alpha is pre-corrected
        
        # Composite
        # C_acc = C_acc + (1 - alpha_acc) * c * alpha
        # alpha_acc = alpha_acc + (1 - alpha_acc) * alpha
        acc_color = acc_color + (1 - acc_opacity) * c * alpha
        acc_opacity = acc_opacity + (1 - acc_opacity) * alpha
        
        # Early termination if fully opaque
        if acc_opacity >= 0.99:
            break
            
    return acc_color, acc_opacity
```

**Edge Cases and Pitfalls:**
-   **Precision:** Floating-point precision errors in depth buffer comparisons may cause gaps or overlaps between surface and volume boundaries [G].
-   **Bounded vs. Un-bounded Logic:** Incorrectly classifying a volume as un-bounded when it has a geometric limit will cause rays to pass through walls, creating visual artifacts [S1].
-   **Memory Access:** Failure to compact address blocks for volume data leads to poor cache performance on graphics hardware [S1].

## Subtopics

### [Volume on Surface (VoS)](20508-volume-on-surface-vos.md)
A hybrid technique that maps volumetric data onto a user-specified iso-surface, allowing interactive rendering of internal details using conventional surface rendering algorithms. It differs by pre-computing ray contributions to the surface faces, whereas Multiple Volume Rendering with Surface Models integrates volumes up to a geometric boundary during the ray casting pass.

## Operating characteristics
-   **Time Complexity:** $O(P \cdot M \cdot N)$, where $P$ is the number of pixels, $M$ is the number of volume sets, and $N$ is the number of samples per ray. The complexity is dominated by the individual ray marching passes for each volume set [S1].
-   **Memory Complexity:** $O(V_{total} + S)$, where $V_{total}$ is the total size of all volume data sets and $S$ is the size of the surface depth buffer [S1].
-   **Performance:** Optimized for graphics hardware by computing contributions per volume individually to maximize memory caching [S1].
-   **Strengths:** Efficiently handles multiple overlapping volumes with clear geometric boundaries; allows differentiation between volume data sets [S1].
-   **Limitations:** Requires pre-computation of the surface depth buffer; limited to scenarios where volumes are bounded by a known surface model [S1].

## Background
The method was developed in the context of architectural visualization, specifically for overlapping and combining 3D voxel volume data with 3D building models [S1]. It was presented by Kreeger et al. in 2012, building on earlier work by Kreeger et al. (1999) regarding surface model integration [S1]. The approach addresses the need for users to differentiate between multiple volume data sets (e.g., thermal, structural, utility) within a complex architectural model [S1].

## Sources
[S1] Volume Rendering in Architecture: Overlapping and combining 3d voxel volume data with 3d building models, https://doi.org/10.52842/conf.ecaade.2012.1.719, fetched 2026-10-26
