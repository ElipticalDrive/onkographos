# Minimum Intensity Projection


## Overview
Minimum Intensity Projection (MinIP) is a volume rendering technique that identifies and renders the voxel with the lowest intensity value along each viewing ray traversing the object [S1]. Unlike standard ray casting which integrates opacity and color, MinIP uses a commutative aggregation function (minimum) to select a single scalar value per pixel, discarding all other depth information [S1][S2, unofficial]. It is primarily used in diagnostic imaging to highlight dark structures such as vessels or bones against brighter backgrounds, offering rapid visualization at the cost of depth perception [S1].

## How it works
**Inputs:**
*   Volume dataset $V$ with dimensions $D_x \times D_y \times D_z$.
*   Ray parameters: start coordinate $C_{start}$, step vector $\Delta C$, and number of steps $N$.
*   Transfer function $T$ mapping scalar values to color/opacity.
*   Material opacity scalar $u_{opacity}$.
*   Transformation matrices: world transform $M_{world}$, camera transform $M_{cam}$, projection transform $M_{proj}$.

**Outputs:**
*   Pixel color $C_{out}$ in physical (linear) color space.
*   Pixel depth $Z_{ndc}$ in normalized device coordinates.
*   Coordinate $C_{min}$ of the minimum intensity voxel.

**Data Structures:**
*   `the_ref`: Scalar accumulator for the minimum intensity found so far. Initialized to $+\infty$ (or a value larger than any possible data value, e.g., 999999.0) [S2, unofficial].
*   `the_coord`: Vector storing the 3D coordinates of the current minimum [S2, unofficial].
*   `the_value`: Vector storing the sampled RGBA value at `the_coord` [S2, unofficial].

**Algorithm:**

1.  **Primary Search:** Iterate $i$ from $0$ to $N-1$.
    *   Calculate current coordinate: $C_i = C_{start} + i \cdot \Delta C$.
    *   Sample volume: $V_i = \text{sample}(C_i, V)$.
    *   Extract scalar reference: $r_i = V_i.r$.
    *   If $r_i < \text{the\_ref}$:
        *   $\text{the\_ref} \leftarrow r_i$
        *   $\text{the\_coord} \leftarrow C_i$
        *   $\text{the\_value} \leftarrow V_i$

2.  **Refinement:** To improve positional accuracy of the minimum, perform a binary search-like refinement.
    *   Initialize `substep` = $\Delta C$.
    *   Iterate $j$ from $0$ to $3$ (4 iterations):
        *   `substep` $\leftarrow$ `substep` $\times 0.5$.
        *   Calculate test coordinates: $C_1 = \text{the\_coord} - \text{substep}$, $C_2 = \text{the\_coord} + \text{substep}$.
        *   Sample: $V_1 = \text{sample}(C_1, V)$, $V_2 = \text{sample}(C_2, V)$.
        *   Extract references: $r_1 = V_1.r$, $r_2 = V_2.r$.
        *   If $r_1 \le \text{the\_ref}$:
            *   $\text{the\_ref} \leftarrow r_1$
            *   $\text{the\_coord} \leftarrow C_1$
            *   $\text{the\_value} \leftarrow V_1$
        *   Else if $r_2 < \text{the\_ref}$:
            *   $\text{the\_ref} \leftarrow r_2$
            *   $\text{the\_coord} \leftarrow C_2$
            *   $\text{the\_value} \leftarrow V_2$

3.  **Color Mapping:**
    *   Map value to color: $C_{raw} = T(\text{the\_value})$.
    *   Convert to physical (linear) color space:
        *   If color space is 'srgb': $C_{phys} = \text{srgb2linear}(C_{raw}.rgb)$.
        *   Else: $C_{phys} = C_{raw}.rgb$.
    *   Calculate opacity: $\alpha = C_{raw}.a \times u_{opacity}$.
    *   Final color: $C_{out} = (C_{phys}, \alpha)$.

4.  **Depth Calculation:**
    *   Convert texture coordinate to data position: $P_{data} = \text{the\_coord} \times D - (0.5, 0.5, 0.5)$.
    *   Transform to world: $P_{world} = M_{world} \times (P_{data}, 1.0)$.
    *   Transform to NDC: $P_{ndc} = M_{proj} \times M_{cam} \times P_{world}$.
    *   Depth: $Z_{ndc} = P_{ndc}.z / P_{ndc}.w$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def minip_render(start_coord, step_coord, nsteps, volume, sizef, transfer_func, 
                 world_transform, cam_transform, proj_transform, colorspace='srgb', 
                 material_opacity=1.0):
    """
    Renders a single ray using Minimum Intensity Projection.
    """
    # Initialization
    the_ref = float('inf')
    the_coord = start_coord
    the_value = None
    
    # Primary loop: coarse search for minimum
    for i in range(nsteps):
        coord = start_coord + i * step_coord
        value = sample_volume(coord, volume, sizef)
        reff = value.r
        
        if reff < the_ref:
            the_ref = reff
            the_coord = coord
            the_value = value
            
    # Refinement loop: binary search for precise minimum location
    substep_coord = step_coord
    for _ in range(4):
        substep_coord = substep_coord * 0.5
        coord1 = the_coord - substep_coord
        coord2 = the_coord + substep_coord
        
        value1 = sample_volume(coord1, volume, sizef)
        value2 = sample_volume(coord2, volume, sizef)
        
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
            
    # Color mapping
    color = transfer_func(the_value)
    
    if colorspace == 'srgb':
        physical_color = srgb_to_linear(color.rgb)
    else:
        physical_color = color.rgb
        
    opacity = color.a * material_opacity
    out_color = vec4(physical_color, opacity)
    
    # Depth calculation
    data_pos = the_coord * sizef - vec3(0.5, 0.5, 0.5)
    world_pos = world_transform * vec4(data_pos, 1.0)
    ndc_pos = proj_transform * cam_transform * world_pos
    depth = ndc_pos.z / ndc_pos.w
    
    return out_color, depth, the_coord

def sample_volume(coord, volume, sizef):
    """
    [G] Standard trilinear interpolation with boundary clamping.
    """
    # Normalize to [0, 1] range assuming volume is stored in [0, sizef-1]
    # Note: Exact sampling implementation depends on the specific volume 
    # data structure and interpolation library used. This is a placeholder 
    # for the core sampling logic required by the algorithm.
    pass

def srgb_to_linear(color_rgb):
    """
    [G] Converts sRGB components to linear RGB.
    """
    linear = []
    for c in color_rgb:
        if c <= 0.04045:
            linear.append(c / 12.92)
        else:
            linear.append(((c + 0.055) / 1.055) ** 2.4)
    return vec3(linear)
```

**Edge Cases and Pitfalls:**
*   **Initialization:** `the_ref` must be initialized to a value strictly greater than any possible intensity in the volume (e.g., $+\infty$ or max uint16/float32) to ensure the first sample updates the minimum [S2, unofficial].
*   **Refinement Direction:** The refinement step checks neighbors in both directions ($-\Delta$ and $+\Delta$). If the minimum lies exactly between grid points, the discrete sampling may oscillate or settle on one side; the 4-iteration binary search provides a heuristic approximation [S2, unofficial].
*   **Depth Loss:** Because only the minimum value is stored, the resulting image lacks inherent depth cues. Interactive exploration or animation is recommended to mitigate this [S1].

## Operating characteristics
*   **Time Complexity:** $O(N)$ per ray, where $N$ is the number of samples. The refinement step adds a constant overhead ($4 \times 2$ samples).
*   **Memory Complexity:** $O(1)$ per pixel for accumulators (`the_ref`, `the_coord`, `the_value`).
*   **Performance:** MinIP is computationally inexpensive and can be accelerated by graphics hardware [S1]. It is significantly faster than full volume rendering with alpha blending because it avoids per-sample compositing and early termination based on opacity.
*   **Strengths:** Rapid generation of images highlighting dark structures (vessels, bones); parallelizable due to commutative aggregation [S1].
*   **Limitations:** Loss of depth information; only a small fraction of data (the minimum) is used, potentially obscuring context [S1].

## Background
MinIP originated in projective diagnostic imaging as a method to clearly represent vessels or bones by selecting the minimum intensity along viewing rays [S1]. It is widely used in 3D CT and MR angiography alongside Maximum Intensity Projection (MIP) [S1]. While static MinIPs lack depth perception, they are effective for interactive exploration or animation sequences [S1].

## Sources
[S1] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S2] https://github.com/pygfx/pygfx/blob/main/pygfx/renderers/wgpu/wgsl/volume_ray.wgsl, https://github.com/pygfx/pygfx/blob/main/pygfx/renderers/wgpu/wgsl/volume_ray.wgsl, fetched 2026-10-29 (unofficial)
