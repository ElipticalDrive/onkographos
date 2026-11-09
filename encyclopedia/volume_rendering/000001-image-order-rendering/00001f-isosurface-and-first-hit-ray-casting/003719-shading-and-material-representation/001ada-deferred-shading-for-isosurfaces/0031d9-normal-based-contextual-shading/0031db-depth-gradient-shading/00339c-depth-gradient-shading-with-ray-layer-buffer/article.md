# Depth-Gradient Shading with Ray-Layer Buffer
Also called: Gradient estimation with ray-layer buffer

## Overview
A technique to estimate surface gradients for shading by analyzing depth coherency in a ray-layer buffer, solving the problem of noisy shading in regions with low volume gradient magnitudes [S1]. It sets itself apart from standard depth-gradient shading by leveraging the layer structure of the ray-casting buffer to distinguish between different surface layers, ensuring that gradient estimation is restricted to neighbors belonging to the same material layer [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray-layer buffer containing depth values and material IDs for each pixel and layer; volume gradient field; pixel spacing in world space ($\Delta x, \Delta y$); gradient magnitude threshold ($T$).
*   **Output:** Blended surface gradient vector for shading.

**Data Structures:**
The ray-layer buffer is a 3D texture or array indexed by $(x, y, l)$, where $x, y$ are pixel coordinates and $l$ is the layer index. Each entry stores:
*   `depth`: The depth value of the surface intersection for this layer.
*   `materialID`: The identifier of the material/surface layer.

**Steps:**
1.  **Assumption:** Depth values produced by the same surface exhibit strong coherency. An entry in the ray-layer buffer has at least one direct neighbor produced by the same surface and encoded into the same layer [S1].
2.  **Neighbor Retrieval:** For the current pixel $(x, y)$ and current layer $l$ being ray-casted, retrieve the depth values from the four immediate neighboring pixels $(x\pm1, y)$ and $(x, y\pm1)$ in the ray-layer buffer for the *same* layer $l$ [S1].
3.  **Difference Calculation:** Compute the absolute difference between the current depth $d_{curr}$ and each neighbor's depth $d_{neigh}$:
    $$ \delta_i = |d_{curr} - d_{neigh}| $$
    where $i \in \{1, 2, 3, 4\}$ corresponds to the four neighbors [S1].
4.  **Selection:** Select the neighbor with the minimal absolute depth difference:
    $$ \delta_{min} = \min(\delta_1, \delta_2, \delta_3, \delta_4) $$
    This selection avoids using depth values from different layer surfaces, as differences between entries with different material IDs are typically huge [S1].
5.  **Surface Gradient Calculation:** Calculate the surface gradient magnitude using the selected depth difference and pixel spacing. Assuming isotropic X and Y directions with pixel spacing $p$:
    $$ g_{surface} = \frac{\delta_{min}}{p} $$
    The direction of the surface gradient is inferred from the direction of the neighbor with the minimal difference [S1].
6.  **Blending:** Blend the surface gradient with the volume gradient. Let $g_{vol}$ be the magnitude of the volume gradient at the current point. Calculate the blend factor:
    $$ b = \min(1, \frac{g_{vol}}{T}) $$
    The final gradient magnitude is:
    $$ g_{final} = b \cdot g_{vol} + (1 - b) \cdot g_{surface} $$
    The final gradient vector is constructed using the direction of the surface gradient (from step 5) scaled by $g_{final}$, or a combination of volume and surface directions weighted by their magnitudes. The source specifies the magnitude blending formula explicitly [S1].

**Parameters:**
*   $T$: Gradient magnitude threshold. Typical values are not given in the sources.
*   $p$: Pixel spacing in world space.

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** Neighbors at the image boundary must be handled (e.g., clamping or ignoring). Not given in the sources.
*   **Discontinuous Surfaces:** If a surface is discontinuous or has large depth jumps, the minimal difference might still pick a neighbor from a different layer if material IDs are not checked. The method assumes material ID encoding prevents this [S1].
*   **Noise:** In regions with very low volume gradient magnitudes, the volume gradient is unreliable, making the surface gradient estimation critical [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def estimate_gradient(ray_layer_buffer, volume_gradient, pixel_spacing, threshold):
    """
    ray_layer_buffer: 3D array [x, y, layer] storing (depth, material_id)
    volume_gradient: Vector field [x, y, layer] storing (gx, gy, gz)
    pixel_spacing: float, distance between neighboring pixels in world space
    threshold: float, gradient magnitude threshold
    """
    # Assume current pixel (x, y) and layer l are provided
    # Get current entry
    current_entry = ray_layer_buffer[x, y, l]
    current_depth = current_entry.depth
    current_material_id = current_entry.material_id
    
    # Get volume gradient magnitude
    vol_grad = volume_gradient[x, y, l]
    vol_grad_mag = length(vol_grad)
    
    # Find neighbor with minimal depth difference in the same layer
    min_diff = infinity
    best_neighbor_dir = None
    
    neighbors = [(x+1, y), (x-1, y), (x, y+1), (x, y-1)]
    
    for nx, ny in neighbors:
        # Check boundary conditions
        if not (0 <= nx < width and 0 <= ny < height):
            continue
            
        neighbor_entry = ray_layer_buffer[nx, ny, l]
        
        # Ensure same material ID to avoid cross-layer gradients
        if neighbor_entry.material_id != current_material_id:
            continue
            
        diff = abs(current_depth - neighbor_entry.depth)
        
        if diff < min_diff:
            min_diff = diff
            # Determine direction from current pixel to neighbor
            dx = nx - x
            dy = ny - y
            best_neighbor_dir = normalize((dx, dy, 0)) # Direction in XY plane
            
    # Calculate surface gradient magnitude
    if min_diff == infinity:
        # No valid neighbor found, use only volume gradient
        surface_grad_mag = 0.0
        surface_grad_dir = (0, 0, 0)
    else:
        surface_grad_mag = min_diff / pixel_spacing
        surface_grad_dir = best_neighbor_dir
        
    # Blend gradients
    blend_factor = min(1.0, vol_grad_mag / threshold)
    
    # Final gradient magnitude
    final_grad_mag = blend_factor * vol_grad_mag + (1.0 - blend_factor) * surface_grad_mag
    
    # Construct final gradient vector
    # Note: The source specifies magnitude blending. Direction handling is implied.
    # If surface gradient is used, its direction is best_neighbor_dir.
    # If volume gradient is used, its direction is vol_grad.
    # A simple implementation might use surface direction if surface_grad_mag > 0, else volume direction.
    if surface_grad_mag > 0:
        final_grad_vec = normalize(surface_grad_dir) * final_grad_mag
    else:
        final_grad_vec = normalize(vol_grad) * final_grad_mag if vol_grad_mag > 0 else (0, 0, 0)
        
    return final_grad_vec
```

## Subtopics

### [Deferred Shading with Approximate Normals](11342-deferred-shading-with-approximate-normals.md)
A technique to calculate view-dependent lighting in layered rendering by storing normal vectors per fragment and performing shading in a deferred step before compositing, rather than multiplying shading coefficients with fragment color before warping.

## Operating characteristics
*   **Time Complexity:** O(1) per pixel per layer, as it involves a constant number of texture lookups and arithmetic operations.
*   **Memory Complexity:** O(N) where N is the number of pixels, as it requires storing the ray-layer buffer.
*   **Measured Performance:** Implemented on Nvidia Quadro FX 5600 using OpenGL 2.1 and GLSL [S1].
*   **Strengths:** Reduces noise in regions with low volume gradient magnitudes; provides shape hints for mask surfaces [S1].
*   **Limitations:** Relies on the assumption of depth coherency; may fail if surfaces are highly discontinuous or if material IDs are not properly encoded [S1].
*   **When to use:** When shading noisy regions due to low volume gradients; when mask surfaces need to be visible and shaded [S1].
*   **When not to use:** When high precision is required for surface normals; when material IDs are not available or reliable [S1].

## Background
Originated in the paper "Multi-layer Volume Ray Casting on GPU" [S1]. The method addresses the issue of noisy shading in regions where the volume gradient magnitude is low, which can occur when surfaces of a mask cut through such regions [S1]. It was implemented as part of a multi-layer volume ray casting system using OpenGL 2.1 and GLSL [S1].

## Sources
[S1] Multi-layer Volume Ray Casting on GPU, https://doi.org/10.2312/vg/vg10/005-012, fetched 2026-10-26
