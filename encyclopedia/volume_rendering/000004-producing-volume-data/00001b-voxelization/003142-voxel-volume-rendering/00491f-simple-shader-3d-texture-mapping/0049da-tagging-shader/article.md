# Tagging Shader

Also called: vzTMTagShader

## Overview
The Tagging Shader performs volumetric tagging by using a second volume as a 3D stencil buffer to selectively render voxels from a primary data volume [S1]. It masks out arbitrary regions in a dataset without modifying the original volume data [S1]. Unlike [Simple Shader (3D Texture Mapping)](12431-simple-shader-3d-texture-mapping.md), which renders the entire volume, this method uses a separate tag texture to control visibility [S1].

## How it works
**Inputs and Outputs:**
*   **Input 1:** `volume` (`vzParameterVolumeTexture`): The primary 3D texture containing the actual volumetric data [S1].
*   **Input 2:** `tag` (`vzParameterVolumeTexture`): A 3D texture acting as the stencil buffer, perfectly overlapping with the `volume` texture [S1].
*   **Input 3:** `lookup_table` (`vzParameterLookupTable`): The transfer function used for color and opacity mapping of the volume data [S1].
*   **Output:** A rendered image where voxels are visible only if their corresponding tag value permits it [S1].

**Data Structures:**
*   **Volume Texture:** Standard 3D texture format [S1].
*   **Tag Texture:** Ideally requires 1 bit per texel, but most graphics hardware allocates at least 1 byte per texel [S1]. The values act as binary thresholds [S1].

**Algorithm Steps:**
1.  Sample the `tag` texture at the current 3D coordinate to obtain the tag value $T$ [S1].
2.  Compare $T$ against the threshold $0.5$: if $T > 0.5$, proceed to render the corresponding texel from the `volume` texture; otherwise, mask out the fragment [S1].
3.  If the fragment is not masked out, sample the `volume` texture at the same coordinate to retrieve the data value $V$ [G].
4.  Apply the `lookup_table` to $V$ to determine the final color and opacity [S1].
5.  Composite the result into the framebuffer [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vzTMTagShader(volume_tex, tag_tex, lookup_table, coord_3d):
    """
    Renders a voxel if its tag value is greater than 0.5.
    
    Args:
        volume_tex: 3D texture object for volume data.
        tag_tex: 3D texture object for tagging/stencil.
        lookup_table: Transfer function object.
        coord_3d: 3D coordinate (x, y, z) in texture space.
        
    Returns:
        Color and opacity, or None if masked out.
    """
    # Step 1: Sample the tag texture
    tag_value = tag_tex.sample(coord_3d)
    
    # Step 2: Check threshold
    if tag_value <= 0.5:
        # Mask out the fragment
        return None
    
    # Step 3: Sample the volume data at the same coordinate (not specified in sources) [G]
    volume_value = volume_tex.sample(coord_3d)
    
    # Step 4: Apply lookup table
    color, opacity = lookup_table.apply(volume_value)
    
    # Step 5: Composite the result into the framebuffer (not specified in sources) [G]
    return (color, opacity)
```

**Edge Cases and Pitfalls:**
*   **Hardware Precision:** On systems lacking `ARB_fragment_program` or `ATI_fragment_shader` OpenGL extensions, the shader falls back to using the hardware stencil buffer [S1]. In this case, the application must ensure the appropriate visual is selected to handle stencil operations correctly [S1].
*   **Memory Overhead:** The tag volume requires additional texture memory [S1]. Since hardware typically uses at least 1 byte per texel for the tag instead of 1 bit, this incurs a significant memory cost [S1].
*   **Alignment:** The `tag` and `volume` textures must be perfectly overlapping in spatial coordinates; misalignment results in incorrect masking [S1].

## Subtopics

### [Mask-based Voxel Filtering](13669-mask-based-voxel-filtering.md)
A fragment shader that combines an original volume dataset with a texture mask to discard voxels or insert new material. It serves as the final composition stage in a volume sculpting pipeline. Unlike the Tagging Shader, which acts as a 3D stencil buffer to tag voxels, this shader performs the actual visual filtering and material replacement during the drawing phase.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels rendered, plus the overhead of sampling the additional tag texture [G].
*   **Memory Complexity:** Requires storage for two 3D textures: the volume data and the tag data [S1]. The tag data typically consumes at least 1 byte per voxel, regardless of its binary nature [S1].
*   **Strengths:** Avoids modifying the original volume data, allowing the same dataset to be used with different masks [S1].
*   **Limitations:** Higher memory usage due to the second 3D texture [S1]. Dependent on specific OpenGL extensions for optimal performance; falls back to stencil buffer on older hardware, which may have performance or configuration constraints [S1].
*   **When to use:** When dynamic masking of volume regions is required without altering the source data [S1].
*   **When not to use:** When memory is constrained, or when the masking pattern is static and can be baked into the volume data itself by setting opacity to zero [S1].

## Background
The Tagging Shader (`vzTMTagShader`) is part of the SGI OpenGL Volumizer library, specifically associated with the Texture Mapping Render Action (TMRenderAction) [S1]. It was available in versions prior to OpenGL Volumizer 2.1 [S1].

## Sources
[S1] Chapter 4. Texture Mapping Render Action, https://techpubs.jurassic.nl/library/manuals/4000/007-4389-010/sgi_html/ch04.html, fetched 2026-10-27
