# Minimum Intensity Projection


## Overview
Minimum Intensity Projection (MinIP) is a volume rendering technique that identifies and renders the voxel with the lowest intensity value along each viewing ray [S1][S2, unofficial]. Unlike standard front-to-back compositing which blends all samples, MinIP discards all but the single minimum sample, effectively highlighting dark structures such as air-filled spaces, vessels in angiography, or low-density tissues against a brighter background [S1]. It is the inverse counterpart to Maximum Intensity Projection (MIP) [S1].

## How it works
MinIP operates as a raycasting mode that searches for the global minimum scalar value along the ray path [S2, unofficial]. It does not use the standard optical compositing model (transmittance/opacity accumulation) because it selects a single voxel rather than blending a volume [S2, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   `volume`: 3D scalar field (texture).
    *   `start_coord`: Ray origin in texture coordinates.
    *   `step_coord`: Ray step vector in texture coordinates.
    *   `nsteps`: Number of discrete samples along the ray.
    *   `sizef`: Volume dimensions (width, height, depth) used for coordinate normalization.
    *   `u_material`: Material properties including opacity multiplier and color space setting.
*   **Outputs**:
    *   `out_color`: Final pixel color (RGB) and opacity (A).
    *   `out.coord`: The 3D texture coordinate of the minimum voxel.
    *   `out.depth`: The normalized device coordinate (NDC) depth ($z/w$) for the minimum voxel.

### Data Structures
The algorithm maintains three state variables during the ray traversal:
1.  `the_ref`: The minimum intensity value found so far (scalar).
2.  `the_coord`: The 3D texture coordinate corresponding to `the_ref` (vec3).
3.  `the_value`: The full sample data (e.g., RGBA vec4) at `the_coord` [S2, unofficial].

### Algorithm Steps
1.  **Initialization**: Set `the_ref` to positive infinity (or a sufficiently large value, e.g., 999999.0). Set `the_coord` to `start_coord`. Initialize `the_value` to zero [S2, unofficial].
2.  **Primary Coarse Search**: Iterate `iter` from 0 to `nsteps - 1`:
    *   Calculate sample coordinate: $coord = start\_coord + iter \cdot step\_coord$.
    *   Sample the volume: $value = \text{sample\_vol}(coord, sizef)$.
    *   Extract scalar reference: $reff = value.r$ (assuming scalar is stored in the red channel) [S2, unofficial].
    *   Update minimum: If $reff < the\_ref$, update $the\_ref = reff$, $the\_coord = coord$, and $the\_value = value$ [S2, unofficial].
3.  **Refinement Loop**: To improve positional accuracy of the minimum, perform a binary search-like refinement:
    *   Initialize `substep_coord` = `step_coord`.
    *   Iterate `iter2` from 0 to 3 (4 iterations):
        *   Halve the step: $substep\_coord = substep\_coord \cdot 0.5$.
        *   Define candidate coordinates: $coord1 = the\_coord - substep\_coord$ and $coord2 = the\_coord + substep\_coord$.
        *   Sample candidates: $value1 = \text{sample\_vol}(coord1, sizef)$ and $value2 = \text{sample\_vol}(coord2, sizef)$.
        *   Extract references: $ref1 = value1.r$ and $ref2 = value2.r$.
        *   Update minimum:
            *   If $ref1 \le the\_ref$ (deliberate $\ge$ in source logic for max, but here checking for min, so $\le$ is correct for finding a new minimum or maintaining equality), update state with $ref1, coord1, value1$ [S2, unofficial].
            *   Else if $ref2 < the\_ref$, update state with $ref2, coord2, value2$ [S2, unofficial].
4.  **Color Mapping**:
    *   Map the scalar value to color: $color = \text{sampled\_value\_to\_color}(the\_value)$.
    *   Convert to physical (linear) color space:
        *   If `colorspace` is 'srgb': $physical\_color = \text{srgb2physical}(color.rgb)$.
        *   Else: $physical\_color = color.rgb$.
    *   Calculate final opacity: $opacity = color.a \cdot u\_material.opacity$.
    *   Set output color: $out\_color = vec4(physical\_color, opacity)$ [S2, unofficial].
5.  **Depth and Coordinate Output**:
    *   Convert texture coordinate to data position: $data\_pos = the\_coord \cdot sizef - vec3(0.5, 0.5, 0.5)$.
    *   Transform to world space: $world\_pos = u\_wobject.world\_transform \cdot vec4(data\_pos, 1.0)$.
    *   Transform to NDC: $ndc\_pos = u\_stdinfo.projection\_transform \cdot u\_stdinfo.cam\_transform \cdot world\_pos$.
    *   Set output depth: $out.depth = ndc\_pos.z / ndc\_pos.w$.
    *   Set output coordinate: $out.coord = the\_coord$ [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def minip_raycast(start_coord, step_coord, nsteps, volume_sampler, sizef, material):
    """
    Minimum Intensity Projection raycasting.
    volume_sampler(coord, sizef) returns a vec4 (rgba) where r is the scalar intensity.
    """
    # 1. Initialization
    the_ref = float('inf')
    the_coord = start_coord
    the_value = vec4(0.0)
    
    # 2. Primary Coarse Search
    for iter in range(nsteps):
        coord = start_coord + iter * step_coord
        value = volume_sampler(coord, sizef)
        reff = value.r
        
        if reff < the_ref:
            the_ref = reff
            the_coord = coord
            the_value = value
            
    # 3. Refinement Loop (4 iterations)
    substep_coord = step_coord
    for iter2 in range(4):
        substep_coord = substep_coord * 0.5
        coord1 = the_coord - substep_coord
        coord2 = the_coord + substep_coord
        
        value1 = volume_sampler(coord1, sizef)
        value2 = volume_sampler(coord2, sizef)
        
        ref1 = value1.r
        ref2 = value2.r
        
        # Note: Source uses >= for max, here we use <= for min to find smaller values
        if ref1 <= the_ref:
            the_ref = ref1
            the_coord = coord1
            the_value = value1
        elif ref2 < the_ref:
            the_ref = ref2
            the_coord = coord2
            the_value = value2
            
    # 4. Color Mapping
    color = colormap(the_value) # Helper: maps scalar to RGBA
    
    if material.colorspace == 'srgb':
        physical_color = srgb_to_linear(color.rgb)
    else:
        physical_color = color.rgb
        
    opacity = color.a * material.opacity
    out_color = vec4(physical_color, opacity)
    
    # 5. Depth and Coordinate Calculation
    data_pos = the_coord * sizef - vec3(0.5, 0.5, 0.5)
    world_pos = material.world_transform * vec4(data_pos, 1.0)
    ndc_pos = material.projection_transform * material.cam_transform * world_pos
    
    out_coord = the_coord
    out_depth = ndc_pos.z / ndc_pos.w
    
    return out_color, out_coord, out_depth
```

### Parameters and Edge Cases
*   **`nsteps`**: Determines the resolution of the initial search. Too few steps may miss the true minimum. [S2, unofficial]
*   **Refinement Iterations**: Fixed at 4 in the reference implementation, providing a $1/16$th precision improvement over the initial step size. [S2, unofficial]
*   **Scalar Channel**: The implementation assumes the scalar intensity is stored in the red channel (`value.r`). [S2, unofficial]
*   **Infinity Initialization**: `the_ref` must be initialized to a value larger than any possible intensity in the volume. [S2, unofficial]
*   **No Compositing**: Since only one voxel is selected, standard opacity accumulation and early ray termination are not applicable. The opacity is derived from the colormap alpha and material settings. [S2, unofficial]

## Operating characteristics
*   **Time Complexity**: $O(N)$ per ray, where $N$ is the number of steps. The refinement loop adds a constant overhead ($4 \times 2$ samples).
*   **Memory Complexity**: $O(1)$ auxiliary memory per ray (storing only the current best reference, coordinate, and value).
*   **Strengths**: Rapidly highlights low-intensity structures (e.g., lungs, vessels in contrast-enhanced scans where background is bright) [S1]. Hardware acceleration is feasible due to simple comparison logic [S1].
*   **Limitations**: Loses all depth information for the selected voxel, making it difficult to perceive the 3D structure of the scene [S1]. Static images lack depth cues. Only a small fraction of the data is used, potentially discarding relevant contextual information [S1].
*   **When to Use**: Angiography, bone visualization (if inverted), or highlighting air-filled spaces. [S1]
*   **When Not to Use**: When depth perception is critical or when the volume contains mixed densities where the minimum is not the feature of interest. [S1]

## Background
MinIP was developed alongside MIP for projective diagnostic imaging, particularly in 3D CT and MR angiography [S1]. It allows for the rapid generation of images that clearly display vessels, tumors, or bones by selecting the extremal intensity values along the ray [S1]. While static MinIPs suffer from a lack of depth information, they are useful in interactive exploration or animation sequences where the drawback is less obvious [S1].

## Sources
[S1] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S2] https://github.com/pygfx/pygfx/blob/main/pygfx/renderers/wgpu/wgsl/volume_ray.wgsl, https://github.com/pygfx/pygfx/blob/main/pygfx/renderers/wgpu/wgsl/volume_ray.wgsl, fetched 2026-10-29 (unofficial)
