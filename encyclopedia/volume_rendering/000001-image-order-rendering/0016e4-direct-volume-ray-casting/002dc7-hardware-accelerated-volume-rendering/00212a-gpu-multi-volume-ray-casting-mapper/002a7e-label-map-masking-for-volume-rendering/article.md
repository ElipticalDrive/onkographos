# Label Map Masking for Volume Rendering

Also called: BinaryMaskType, LabelMapMaskType, MaskInput

## Overview
Label Map Masking is a technique that confines volume rendering to specific regions using either a binary mask or a multi-label map, supporting blending between standard and mask-specific color transfer functions [S1, unofficial]. It sets itself apart from sibling mappers by explicitly defining how mask values (binary or labeled) dictate the selection of color transfer functions (CTFs) and their blending weights, rather than simply acting as a visibility gate [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A primary volume dataset (typically `vtkImageData`) and a mask dataset (`MaskInput`). The mask must be of type `UCHAR` (unsigned char) [S1, unofficial].
*   **Output:** A rendered image where pixel colors are determined by the volume data and the mask configuration [S1, unofficial].

**Data Structures:**
*   **Mask Data:** A 3D grid of `unsigned char` values.
    *   For **Binary Mask**: Values are strictly `0` (outside) or `255` (inside) [S1, unofficial].
    *   For **Label Map**: Values are restricted to exactly three labels: `0`, `1`, and `2` [S1, unofficial].

**Parameters:**
*   **MaskInput** (`vtkImageData*`): The source of the mask data [S1, unofficial].
*   **MaskBlendFactor** (`float`): Clamped to $[0.0, 1.0]$. Default is $1.0$. Controls the blending weight between the standard CTF and the mask-specific CTF [S1, unofficial].
    *   $0.0$: Uses only the standard CTF.
    *   $1.0$: Uses only the mask-specific CTF [S1, unofficial].

**Algorithm Steps:**
1.  **Determine Mask Type:** Check the number of unique non-zero values in the mask or the configuration of `MaskInput` to determine if it is a Binary Mask or a Label Map [S1, unofficial].
2.  **Sample Volume Data:** For each ray cast, sample the primary volume data at the current position to obtain the scalar value $v$ [S1, unofficial].
3.  **Sample Mask Data:** Retrieve the mask value $m$ at the corresponding spatial position [S1, unofficial].
4.  **Apply Mask Logic:**
    *   **If Binary Mask:**
        *   If $m = 255$, proceed with rendering using the configured CTF logic [S1, unofficial].
        *   If $m = 0$, the region is outside the mask; rendering is confined strictly to regions where $m=255$ [S1, unofficial].
    *   **If Label Map:**
        *   Identify the label $L \in \{0, 1, 2\}$ from $m$ [S1, unofficial].
        *   Retrieve the standard CTF output $C_{std}$ for value $v$ [S1, unofficial].
        *   Retrieve the mask-specific CTF output $C_{mask}$ for value $v$ [S1, unofficial].
        *   Calculate the final color $C_{final}$ based on $L$:
            *   If $L = 0$: $C_{final} = C_{std}$ [S1, unofficial].
            *   If $L = 1$: $C_{final} = \text{Blend}(C_{std}, C_{mask}, \text{MaskBlendFactor})$ [S1, unofficial].
            *   If $L = 2$: $C_{final} = \text{Blend}(C_{std}, C_{mask}, \text{MaskBlendFactor})$ [S1, unofficial].
5.  **Composite:** Perform standard volume ray casting compositing (e.g., front-to-back alpha blending) using $C_{final}$ and the opacity derived from the CTFs [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_label_map_masking(volume_data, mask_data, mask_blend_factor, standard_ctf, mask_ctf):
    """
    Renders volume data using label map masking.
    
    Args:
        volume_data: 3D array of scalar values.
        mask_data: 3D array of unsigned char values (0, 1, or 2).
        mask_blend_factor: Float in [0.0, 1.0].
        standard_ctf: Function mapping scalar value to (color, opacity).
        mask_ctf: Function mapping scalar value to (color, opacity).
        
    Returns:
        Rendered image buffer.
    """
    # Initialize output buffer
    image_height, image_width, depth = volume_data.shape
    output_image = np.zeros((image_height, image_width, 3))
    
    # Iterate over pixels (simplified for 2D projection example)
    for y in range(image_height):
        for x in range(image_width):
            # Ray casting loop would normally iterate over z (depth)
            # Here we assume a single sample or pre-computed composite for brevity
            # In a real implementation, this is inside the ray casting loop
            
            # Sample volume and mask at current ray position
            # Note: In a full ray caster, 'pos' changes along the ray
            pos = get_current_ray_position()
            v = volume_data[pos]
            m = mask_data[pos]
            
            # Get CTF outputs
            c_std, a_std = standard_ctf(v)
            c_mask, a_mask = mask_ctf(v)
            
            # Determine final color based on mask label
            if m == 0:
                # Label 0: Use standard CTF only
                final_color = c_std
                final_opacity = a_std
            elif m == 1 or m == 2:
                # Labels 1 and 2: Blend standard and mask CTFs
                # Linear interpolation of color and opacity
                # C_final = (1 - factor) * C_std + factor * C_mask
                final_color = (1.0 - mask_blend_factor) * c_std + mask_blend_factor * c_mask
                final_opacity = (1.0 - mask_blend_factor) * a_std + mask_blend_factor * a_mask
            else:
                # Binary mask case: if m != 255, skip or treat as background
                # Assuming binary mask check is separate or m=255 is the only valid "inside"
                # If this is a label map, m should be 0,1,2. If binary, m should be 0,255.
                # This branch handles unexpected values or binary mask "outside"
                final_color = np.zeros(3)
                final_opacity = 0.0
                
            # Composite into image (simplified front-to-back)
            # output_image[y, x] = blend(output_image[y, x], final_color, final_opacity)
            pass
            
    return output_image
```

**Edge Cases and Pitfalls:**
*   **Data Type Mismatch:** The mask must be `UCHAR`. Using other types may lead to undefined behavior or incorrect label interpretation [S1, unofficial].
*   **Label Constraints:** Label maps are restricted to exactly three labels (0, 1, 2). Values outside this range are not defined in the source specification [S1, unofficial].
*   **Precision:** `MaskBlendFactor` is clamped to $[0.0, 1.0]$. Values outside this range should be clamped by the implementation [S1, unofficial].

## Subtopics

### Mask Volume Brushing
A technique using a 3D binary data structure (mask volume) linked between attribute space views and object space rendering to select and highlight features [S2, book]. It differs from standard label map masking by emphasizing an interactive workflow where the mask is created and modified by brushing tensor properties in attribute space, then evaluated for rendering in the spatial domain [S2, book]. It is chosen when interactive feature selection in high-dimensional attribute spaces is required [S2, book]. Mask values are binary (0 or 1) stored in a 3D structure of the same size as the input data [S2, book].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels in the volume and the number of rays cast [S1, unofficial]. The masking operation adds negligible overhead as it involves simple array lookups and conditional logic [S1, unofficial].
*   **Memory Complexity:** Requires additional memory for the mask volume, which is typically the same size as the input volume data [S2, book].
*   **Strengths:** Allows for precise region-specific rendering and blending of visual properties [S1, unofficial]. Supports interactive exploration through mask brushing [S2, book].
*   **Limitations:** Restricted to specific mask types (binary or 3-label) in the primary implementation [S1, unofficial]. Performance may degrade if the mask volume is large and not optimized for cache locality [S1, unofficial].
*   **When to Use:** When specific anatomical or physical regions need to be highlighted with distinct color transfer functions [S1, unofficial]. When interactive feature selection is needed [S2, book].
*   **When Not to Use:** When the mask requires more than 3 labels or non-binary values [S1, unofficial]. When memory is constrained and the mask volume size is prohibitive [S2, book].

## Background
Label Map Masking was introduced as part of the GPU Multi-Volume Ray Casting Mapper in VTK, specifically in the `vtkGPUMultiVolumeRayCastMapper` class [S1, unofficial]. It was developed to support multiple volumes and adaptive rendering with masking capabilities [S1, unofficial]. The concept of using mask volumes for interactive visualization in attribute space was previously explored in the context of stress tensor field analysis [S2, book].

## Sources
[S1] bozorgi/VTKMultiVolumeRayCaster, https://github.com/bozorgi/VTKMultiVolumeRayCaster, fetched 2026-10-26 (unofficial)
[S2] A Visual Approach to Analysis of Stress Tensor Fields, https://doi.org/10.4230/dfu.vol2.sciviz.2011.188, fetched 2026-10-26 (book)
