# Minimum Intensity Projection


## Overview
Minimum Intensity Projection (MinIP) is a volume rendering technique that projects the voxel with the lowest scalar intensity along each viewing ray onto the 2D image plane [S1][S2, unofficial]. Unlike standard direct volume rendering which composites opacity-weighted colors, MinIP discards depth cues to highlight low-intensity structures such as air-filled spaces, vessels, or bone trabeculae [S1]. It is distinguished from its sibling Maximum Intensity Projection (MIP) by inverting the aggregation operator from maximum to minimum, making it complementary for visualizing hypodense or low-signal anatomical features [S1].

## How it works
MinIP builds on the shared ray-casting loop defined in [Direct volume ray casting](19-direct-volume-ray-casting.md). For each pixel, a ray is traversed through the volume, and the scalar value $s_i$ at each sample point is compared against a running minimum. The final pixel color is derived from the sample with the global minimum intensity along that ray [S1][S2, unofficial].

### Inputs and Outputs
*   **Input**: Volume dataset $V$ (scalar field), camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$), transfer function $T$, and rendering parameters (number of steps $N$, initial search bounds).
*   **Output**: A 2D image where each pixel $(u,v)$ contains the color $C$ and depth $Z$ corresponding to the minimum intensity voxel found along the ray.

### Data Structures
The implementation typically maintains the following state per ray:
*   `the_ref`: The current minimum scalar value found, initialized to positive infinity ($\infty$) or the maximum representable float [S2, unofficial].
*   `the_coord`: The 3D coordinate of the voxel holding `the_ref` [S2, unofficial].
*   `the_value`: The full sampled value (e.g., RGBA or vec4) associated with `the_coord` [S2, unofficial].
*   `step_coord`: The displacement vector between consecutive samples, calculated as $(t_{end} - t_{start})/N \cdot \mathbf{d}$ [S2, unofficial].

### Algorithm Steps

1.  **Initialization**: Set `the_ref` to $\infty$ (or a value larger than any possible data value). Set `the_coord` to the ray start position. Set `the_value` to zero or undefined [S2, unofficial].
2.  **Primary Coarse Search**: Iterate $i$ from $0$ to $N-1$:
    *   Calculate sample coordinate: $\mathbf{p}_i = \mathbf{o} + i \cdot \Delta t \cdot \mathbf{d}$ [S2, unofficial].
    *   Sample volume value $v_i = \text{Sample}(\mathbf{p}_i)$ [S2, unofficial].
    *   Extract scalar $s_i = v_i.r$ (or the primary scalar component) [S2, unofficial].
    *   If $s_i < \text{the\_ref}$:
        *   Update $\text{the\_ref} = s_i$
        *   Update $\text{the\_coord} = \mathbf{p}_i$
        *   Update $\text{the\_value} = v_i$ [S2, unofficial].
3.  **Refinement Loop**: To locate the minimum more precisely than the discrete grid allows, perform a binary search-like refinement [S2, unofficial]:
    *   Initialize `substep` = $\Delta t \cdot \mathbf{d}$ (or a fraction thereof) [S2, unofficial].
    *   Repeat for $k = 0$ to $3$ (4 iterations):
        *   `substep` = `substep` / 2 [S2, unofficial].
        *   Calculate candidate coordinates: $\mathbf{c}_1 = \text{the\_coord} - \text{substep}$ and $\mathbf{c}_2 = \text{the\_coord} + \text{substep}$ [S2, unofficial].
        *   Sample values: $v_1 = \text{Sample}(\mathbf{c}_1)$, $v_2 = \text{Sample}(\mathbf{c}_2)$ [S2, unofficial].
        *   Extract scalars: $s_1 = v_1.r$, $s_2 = v_2.r$ [S2, unofficial].
        *   If $s_1 \le \text{the\_ref}$:
            *   Update $\text{the\_ref} = s_1$, $\text{the\_coord} = \mathbf{c}_1$, $\text{the\_value} = v_1$ [S2, unofficial].
        *   Else if $s_2 < \text{the\_ref}$:
            *   Update $\text{the\_ref} = s_2$, $\text{the\_coord} = \mathbf{c}_2$, $\text{the\_value} = v_2$ [S2, unofficial].
        *   *Note: The source uses `>=` for the first check in a max-intensity context, but for MinIP, the logic inverts to `<` and `<=` to find lower values [S2, unofficial]. The provided WGSL snippet for MinIP uses `reff < the_ref` in the main loop and `ref1 <= the_ref` in refinement.*
4.  **Classification and Output**:
    *   Map the final value to color: $C_{raw} = T(\text{the\_value})$ [S2, unofficial].
    *   Convert to physical color space if necessary (e.g., sRGB to linear) [S2, unofficial].
    *   Compute opacity $\alpha = C_{raw}.a \times \text{material\_opacity}$ [S2, unofficial].
    *   Compute depth $Z$ by transforming `the_coord` to world space and then to NDC space [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def minip_render(ray_origin, ray_direction, volume, transfer_function, num_steps):
    """
    Renders a single ray using Minimum Intensity Projection.
    """
    # 1. Initialization
    the_ref = float('inf')
    the_coord = ray_origin
    the_value = None
    
    # Step size vector
    step_coord = ray_direction * (1.0 / num_steps) # Assuming normalized ray length for simplicity
    
    # 2. Primary Coarse Search
    for i in range(num_steps):
        coord = ray_origin + i * step_coord
        value = volume.sample(coord) # Trilinear interpolation assumed
        scalar = value.r # Assuming scalar is in the first component
        
        if scalar < the_ref:
            the_ref = scalar
            the_coord = coord
            the_value = value
            
    # 3. Refinement Loop (4 iterations)
    substep_coord = step_coord
    for _ in range(4):
        substep_coord = substep_coord * 0.5
        
        coord1 = the_coord - substep_coord
        coord2 = the_coord + substep_coord
        
        value1 = volume.sample(coord1)
        value2 = volume.sample(coord2)
        
        ref1 = value1.r
        ref2 = value2.r
        
        if ref1 <= the_ref:
            the_ref = ref1
            the_coord = coord1
            the_value = value1
        elif ref2 < the_ref:
            the_ref = ref2
            the_coord = coord2
            the_value = value2
            
    # 4. Classification
    color_raw = transfer_function(the_value)
    
    # Convert to physical colorspace if needed (simplified)
    if color_raw.colorspace == 'srgb':
        physical_color = srgb_to_linear(color_raw.rgb)
    else:
        physical_color = color_raw.rgb
        
    opacity = color_raw.a * material_opacity
    
    # 5. Output
    depth = compute_depth(the_coord)
    
    return {
        'color': vec4(physical_color, opacity),
        'depth': depth,
        'coord': the_coord
    }
```

### Parameters and Edge Cases
*   **Number of Steps ($N$)**: Determines the precision of the coarse search. Higher $N$ increases accuracy but costs more [S2, unofficial].
*   **Refinement Iterations**: The source uses 4 iterations, halving the step size each time [S2, unofficial].
*   **Initialization Value**: `the_ref` must be initialized to a value strictly greater than any possible data value (e.g., `float('inf')` or `999999.0` as in the source) [S2, unofficial].
*   **Boundary Conditions**: If the refinement step moves outside the volume bounds, sampling should clamp or return a sentinel value that does not incorrectly become the minimum [G].
*   **Precision**: Floating-point comparison order matters. The source uses `<=` for the first refinement check and `<` for the second, ensuring deterministic behavior when equal values are found [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N)$ per ray for the coarse search, plus $O(1)$ for the fixed 4-step refinement. Total complexity is linear with respect to the number of samples per ray [S2, unofficial].
*   **Memory Complexity**: $O(1)$ auxiliary memory per ray, storing only the current best reference, coordinate, and value [S2, unofficial].
*   **Strengths**: Rapid generation of images highlighting low-intensity structures (e.g., airways, vessels in angiography); hardware acceleration is straightforward as it requires only a reduction operation [S1].
*   **Limitations**: Loss of depth information makes it difficult to distinguish overlapping structures; static images lack context compared to volume rendering [S1].
*   **When to Use**: When visualizing hypodense structures like lungs, air-filled sinuses, or specific vessel types in CT/MR angiography [S1].
*   **When Not to Use**: When depth perception is critical or when the volume contains significant noise that could be mistaken for the true minimum [S1].

## Background
MinIP was developed as a complementary technique to Maximum Intensity Projection (MIP) in the context of projective diagnostic imaging [S1]. It gained prominence in 3D CT and MR angiography for its ability to clearly display vessels, tumors, or bones by highlighting the minimum intensity along the ray [S1]. While early implementations were computationally expensive, graphics hardware acceleration has made interactive exploration possible [S1]. Despite its utility, static MinIP images are limited by the lack of depth information, leading to the development of interactive or animated sequences to mitigate this drawback [S1].

## Sources
[S1] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S2] https://github.com/pygfx/pygfx/blob/main/pygfx/renderers/wgpu/wgsl/volume_ray.wgsl, https://github.com/pygfx/pygfx/blob/main/pygfx/renderers/wgpu/wgsl/volume_ray.wgsl, fetched 2026-10-29 (unofficial)
