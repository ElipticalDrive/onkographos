# Minimum Intensity Projection


## Overview
Minimum Intensity Projection (MinIP) is a volume rendering technique that identifies and renders the voxel with the lowest intensity value along each viewing ray [S1][S2, unofficial]. Unlike standard front-to-back compositing which blends all samples, MinIP discards all but the single minimum sample, effectively highlighting dark structures such as air-filled spaces, vessels in angiography, or low-density tissues against a brighter background [S1]. It is the inverse counterpart to Maximum Intensity Projection (MIP) [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset $V$, camera parameters (view and projection matrices), ray origin $O$, ray direction $D$, and step size $\Delta s$.
*   **Output:** A 2D image where each pixel $(u,v)$ contains the color and opacity of the minimum-intensity voxel found along the corresponding ray.

**Data Structures:**
The method maintains three accumulator variables during ray traversal:
1.  `the_ref`: The minimum intensity value found so far. Initialized to positive infinity ($+\infty$) or the maximum representable float value [S2, unofficial].
2.  `the_coord`: The 3D spatial coordinate of the voxel yielding `the_ref`. Initialized to the ray start point [S2, unofficial].
3.  `the_value`: The full sample data (e.g., RGBA or RGB+Intensity) at `the_coord`. Initialized to zero or background [S2, unofficial].

**Algorithm Steps:**

1.  **Ray Initialization:** Calculate the ray step vector $\vec{step} = \Delta s \cdot \vec{D}$ and the starting coordinate $C_{start}$ [S2, unofficial].
2.  **Primary Search Loop:** Iterate $i$ from $0$ to $N_{steps}-1$:
    *   Compute current coordinate: $C_i = C_{start} + i \cdot \vec{step}$.
    *   Sample the volume at $C_i$ to get value $V_i$ and intensity $I_i = V_i.r$ (assuming intensity is stored in the red channel or first component) [S2, unofficial].
    *   If $I_i < \text{the\_ref}$:
        *   $\text{the\_ref} \leftarrow I_i$
        *   $\text{the\_coord} \leftarrow C_i$
        *   $\text{the\_value} \leftarrow V_i$
3.  **Refinement Loop (Optional but recommended for accuracy):** To locate the minimum more precisely between grid points, perform a binary search-like refinement [S2, unofficial]:
    *   Initialize `substep` = $\vec{step}$.
    *   Iterate $j$ from $0$ to $3$ (or until convergence):
        *   `substep` $\leftarrow$ `substep` $\times 0.5$.
        *   Calculate candidate coordinates: $C_1 = \text{the\_coord} - \text{substep}$ and $C_2 = \text{the\_coord} + \text{substep}$.
        *   Sample $V_1$ at $C_1$ and $V_2$ at $C_2$, extracting intensities $I_1$ and $I_2$ [S2, unofficial].
        *   If $I_1 \le \text{the\_ref}$:
            *   $\text{the\_ref} \leftarrow I_1$
            *   $\text{the\_coord} \leftarrow C_1$
            *   $\text{the\_value} \leftarrow V_1$
        *   Else if $I_2 < \text{the\_ref}$:
            *   $\text{the\_ref} \leftarrow I_2$
            *   $\text{the\_coord} \leftarrow C_2$
            *   $\text{the\_value} \leftarrow V_2$
    *   *Note: The source code uses `>=` for the first check in the refinement loop, implying a preference for the left side on ties, but strictly minimizes intensity* [S2, unofficial].
4.  **Color Mapping and Compositing:**
    *   Map the intensity `the_ref` (or full `the_value`) to a color $C_{map}$ using a transfer function [S2, unofficial].
    *   Convert to physical color space if necessary (e.g., sRGB to linear) [S2, unofficial].
    *   Calculate final opacity $\alpha_{final} = C_{map}.a \times \alpha_{material}$, where $\alpha_{material}$ is a user-defined opacity factor [S2, unofficial].
    *   The output color is $C_{out} = (C_{map}.rgb, \alpha_{final})$.
5.  **Depth Calculation:**
    *   Convert `the_coord` to world space and then to NDC to determine the depth buffer value for occlusion testing [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def minip_render(ray_origin, ray_direction, volume, step_size, n_steps):
    """
    Renders a single pixel using Minimum Intensity Projection.
    """
    # 1. Initialization
    the_ref = float('inf')
    the_coord = ray_origin
    the_value = volume.sample(ray_origin)
    
    # Pre-calculate step vector
    step_vec = ray_direction * step_size
    
    # 2. Primary Search Loop
    for i in range(n_steps):
        current_coord = ray_origin + (i * step_vec)
        # Clamp or handle boundary checks if necessary [G]
        value = volume.sample(current_coord)
        intensity = value.r  # Assuming intensity is in the red channel
        
        if intensity < the_ref:
            the_ref = intensity
            the_coord = current_coord
            the_value = value
            
    # 3. Refinement Loop (Binary Search for precision)
    substep = step_vec
    for _ in range(4): # 4 iterations as per source [S2, unofficial]
        substep = substep * 0.5
        coord1 = the_coord - substep
        coord2 = the_coord + substep
        
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
            
    # 4. Color Mapping
    # Assume a generic transfer function exists
    color_mapped = transfer_function(the_value)
    
    # Convert to linear space if needed [S2, unofficial]
    if color_mapped.color_space == 'srgb':
        physical_color = srgb_to_linear(color_mapped.rgb)
    else:
        physical_color = color_mapped.rgb
        
    opacity = color_mapped.a * material_opacity
    
    # 5. Output
    return {
        'color': vec4(physical_color, opacity),
        'depth': calculate_depth(the_coord),
        'coord': the_coord
    }
```

**Parameters:**
*   `n_steps`: Number of samples along the ray. Higher values increase accuracy but cost more time [S2, unofficial].
*   `step_size`: Distance between samples. Determined by volume resolution and camera distance [S2, unofficial].
*   `material_opacity`: Global opacity multiplier [S2, unofficial].

**Edge Cases:**
*   **Flat Regions:** If multiple voxels have the same minimum intensity, the result depends on the order of traversal and the refinement logic (source prefers left-side updates on equality) [S2, unofficial].
*   **Noise:** MinIP is highly sensitive to noise because it selects the absolute minimum. A single noisy low-value voxel can obscure the true structure [S1].
*   **Boundaries:** Ray origins or coordinates falling outside the volume bounds must be handled by the `sample_vol` function, typically by returning a default high value or clamping [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot R)$, where $N$ is the number of pixels and $R$ is the number of samples per ray. The refinement loop adds a constant factor (typically 4 iterations) [S2, unofficial].
*   **Memory Complexity:** $O(1)$ per pixel for accumulators, plus the storage for the volume dataset.
*   **Strengths:** Rapidly highlights dark, tubular, or linear structures (e.g., vessels, airways) by suppressing bright background tissue [S1]. Hardware-accelerated generation is feasible [S1].
*   **Limitations:** Loses all depth information (occlusion is not handled via compositing, only via the final depth buffer if used for sorting) [S1]. Sensitive to noise and artifacts that create artificially low intensity values [S1]. Static images lack depth cues, making interactive exploration or animation sequences preferable [S1].
*   **When to use:** Angiography, CT/MR imaging where dark structures need emphasis, or when visualizing air-filled spaces.
*   **When not to use:** When depth perception is critical, or when the volume contains significant noise that could be mistaken for signal.

## Background
MinIP was developed alongside MIP for projective diagnostic imaging, particularly in 3D CT and MR angiography [S1]. It allows for the clear representation of vessels or bones (if they appear darker than surrounding tissue in specific modalities or windowing) at the cost of depth information [S1]. While static MinIPs suffer from a lack of depth perception, they can be interactively explored or used in animation sequences to mitigate this drawback [S1]. Local MinIP (MinP) and Closest Vessel Projection (CVP) are variations often used in conjunction with slab techniques [S1].

## Sources
[S1] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S2] https://github.com/pygfx/pygfx/blob/main/pygfx/renderers/wgpu/wgsl/volume_ray.wgsl, https://github.com/pygfx/pygfx/blob/main/pygfx/renderers/wgpu/wgsl/volume_ray.wgsl, fetched 2026-10-29 (unofficial)
