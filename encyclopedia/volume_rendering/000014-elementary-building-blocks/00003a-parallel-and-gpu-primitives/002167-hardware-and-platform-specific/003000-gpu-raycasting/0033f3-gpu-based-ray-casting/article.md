# GPU-based Ray Casting


## Overview
GPU-based ray casting is a rendering technique for unstructured grids that appears to sidestep the limitations of visibility orders and path tracing (PT) by avoiding explicit visibility order computation and many PT accuracy issues [S1, unofficial]. It differs from PT by requiring all data to reside in GPU memory, which limits dataset size, whereas PT streams data from CPU memory [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Unstructured mesh data (tetrahedra), scalar fields, and ray origins/directions.
*   **Output:** Accumulated color and opacity per pixel.

**Data Structures:**
*   **Mesh Data:** Stored per tetrahedron in texture memory. Total size: 192 bytes.
    *   Fields: Vertices, normals, connectivity, scalar fields (gradient and reference static value), codebook indices, mean [S2, unofficial].
    *   Scalar data is stored using the gradient of the scalar field and a reference static value to allow reconstruction via interpolation [S2, unofficial].
*   **Textures:**
    *   Two sets of textures: one for current ray intersection information, one for accumulated color [S2, unofficial].
    *   Codebooks: $C_8$ ($8 \times 256$ float) and $C_{64}$ ($64 \times 256$ float) [S2, unofficial].

**Steps:**
1.  Store mesh information (vertices, normals, connectivity, scalar fields) per tetrahedron in texture memory [S2, unofficial].
2.  Use a fragment shader to compute ray intersections, advancing one intersection inside the mesh at a time while evaluating the volume rendering integral [S2, unofficial].
3.  Use a depth-peeling approach to handle non-convex meshes [S2, unofficial].
4.  For time-varying data, store codebooks in GPU memory. Use a paging mechanism to swap codebooks when exceeding 64 time instances to avoid stalls [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Time-Varying GPU-based Ray Casting
# Based on [S2, unofficial]

def render_time_varying_mesh(mesh_data, codebooks, ray_origins, ray_directions):
    """
    Renders time-varying scalar fields on unstructured meshes using GPU ray casting.
    
    Args:
        mesh_data: Dictionary containing per-tetrahedron data (192B each).
        codebooks: List of codebook textures (C8 and C64).
        ray_origins: Array of ray origins.
        ray_directions: Array of ray directions.
        
    Returns:
        Color and opacity buffers.
    """
    # Initialize accumulated color and opacity buffers
    accumulated_color = zeros_like(ray_origins)
    accumulated_opacity = zeros_like(ray_origins)
    
    # Load initial codebooks into GPU memory
    current_codebook = codebooks[0]
    next_codebook = codebooks[1]
    time_index = 0
    
    # Fragment shader execution for each ray
    for ray in zip(ray_origins, ray_directions):
        ray_color = 0.0
        ray_opacity = 0.0
        current_pos = ray.origin
        t = 0.0
        
        # Advance ray through mesh intersections
        while t < max_distance:
            # Find intersection with current tetrahedron
            tet = find_intersecting_tetrahedron(current_pos, ray.direction, mesh_data)
            if tet is None:
                break
                
            # Calculate exit point and step size
            exit_pos, step_size = calculate_exit_point(current_pos, ray.direction, tet)
            
            # Evaluate scalar field at current position
            # Scalar data reconstructed from gradient and reference static value
            scalar_value = reconstruct_scalar(current_pos, tet.scalar_gradient, tet.reference_static_value)
            
            # Map scalar value to color using codebook
            color = map_scalar_to_color(scalar_value, current_codebook)
            
            # Update accumulated color and opacity
            ray_color += color * (1.0 - ray_opacity)
            ray_opacity += (1.0 - ray_opacity) * opacity_factor
            
            # Move to next intersection
            current_pos = exit_pos
            t += step_size
            
            # Check for codebook swap (time-varying)
            if time_index >= 64:
                swap_codebooks(current_codebook, next_codebook)
                time_index = 0
                next_codebook = load_next_codebook()
            else:
                time_index += 1
                
        accumulated_color[ray.id] = ray_color
        accumulated_opacity[ray.id] = ray_opacity
        
    return accumulated_color, accumulated_opacity
```

**Parameters:**
*   Codebook sizes: $C_8$ ($8 \times 256$ float), $C_{64}$ ($64 \times 256$ float) [S2, unofficial].
*   Max time instances per codebook: 64 [S2, unofficial].

**Edge Cases and Pitfalls:**
*   Accuracy limitations due to ray-triangle intersections lower image quality [S1, unofficial].
*   All data must be in GPU memory, limiting dataset size [S1, unofficial].
*   Depth peeling is required for non-convex meshes [S2, unofficial].

## Subtopics

### Time-Varying GPU-based Ray Casting
A GPU-based ray casting algorithm that computes ray intersections using fragment shaders, advancing one intersection inside the mesh at a time while evaluating the volume rendering integral. Extended to handle TVSF by storing mesh data and codebooks in texture memory. It uses a depth-peeling approach to handle non-convex meshes and a paging mechanism to swap codebooks when exceeding 64 time instances to avoid stalls [S2, unofficial]. Per tetrahedron data stored in texture memory includes vertices, normals, connectivity, scalar fields (gradient and reference static value), codebook indices, and mean, totaling 192B [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Requires all data to be in GPU memory, limiting dataset size [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Appears to sidestep many limitations of visibility orders and PT, particularly the explicit need for computing a visibility order and many accuracy issues of PT [S1, unofficial].
*   **Limitations:** Accuracy limitations due to ray-triangle intersections lower image quality [S1, unofficial]. Requires all data in GPU memory, limiting dataset size [S1, unofficial].
*   **When to use:** When avoiding explicit visibility order computation is beneficial and dataset size fits in GPU memory [S1, unofficial].
*   **When not to use:** When dataset size exceeds GPU memory capacity [S1, unofficial].

## Background
GPU-based ray casting implementations can be traced back to original implementations by authors [S1, unofficial]. It is used for volume rendering of unstructured grids [S1, unofficial].

## Sources
[S1] PDF A Survey of GPU-Based Volume Rendering of Unstructured Grids, https://www.sci.utah.edu/~stevec/papers/gputet_survey.pdf, fetched 2026-10-27 (unofficial)
[S2] PDF Volume Rendering of Time-Varying Scalar Fields on Unstructured Meshes, https://www.sci.utah.edu/~stevec/papers/tvrc.pdf, fetched 2026-10-27 (unofficial)
