# 8-bit segment ordinal storage
Also called: 8-bit shadow volume storage, SegmentedVolumeData

## Overview
8-bit segment ordinal storage maps each voxel in a volume to a single integer identifier (0–255) using a compact 8-bit texture, enabling distinct rendering styles or properties per segment [S1, S2, unofficial]. It differs from explicit coordinate lists by offering random access and lower memory overhead for dense segmentations (≥33% coverage), while differing from sparse structures by storing data in a dense grid format [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume grid of dimensions $W \times H \times D$.
*   **Input:** A 3D texture `segmentedVoxels` containing segment identifiers.
*   **Input:** An array of rendering styles `renderStyle`.
*   **Output:** A compressed representation where each voxel $(x,y,z)$ holds an 8-bit segment ID.
*   **Output:** A rendering pipeline that applies the style indexed by the segment ID to the underlying volume data.

**Data Structures:**
1.  **Segment Ordinal Volume:** A dense 3D array (or 3D texture) of 8-bit unsigned integers.
    *   **Layout:** One byte per voxel.
    *   **Size:** $W \times H \times D$ bytes.
    *   **Content:** The value at index $(x,y,z)$ is the segment ID $S_{x,y,z} \in [0, 255]$.
2.  **Rendering Style Array:** A list of style definitions (e.g., opacity maps, color maps).
    *   **Indexing:** The segment ID $S_{x,y,z}$ serves as the direct index into this array.

**Steps:**
1.  **Segmentation:** Assign each voxel in the volume to a segment. If a voxel belongs to multiple segments, this method does not support fractional segmentation; it assumes a single dominant segment [S1, unofficial].
2.  **Encoding:** Store the segment ID for each voxel in the 8-bit `segmentedVoxels` texture.
    *   If the texture has 4 components (RGBA), the Alpha channel stores the segment ID [S2, unofficial].
    *   If the texture has 3 components (RGB), the Blue channel stores the segment ID, while Red and Green store auxiliary 2D data (e.g., coordinates or metadata) [S2, unofficial].
    *   If the texture has 2 components, the specific mapping for the ID is not fully detailed in the provided sources, but the ID must be derivable [S2, unofficial].
3.  **Rendering Lookup:** For each voxel $(x,y,z)$ during ray marching or casting:
    *   Read the segment ID $S_{x,y,z}$ from the `segmentedVoxels` texture.
    *   Check if `segmentEnabled` is true.
    *   If true, retrieve the rendering style $Style_{S_{x,y,z}}$ from the `renderStyle` array using $S_{x,y,z}$ as the index.
    *   If `renderStyle` is empty or undefined, apply a default `OpacityMapVolumeStyle` [S2, unofficial].
    *   Apply $Style_{S_{x,y,z}}$ to the volumetric data at $(x,y,z)$.

**Parameters:**
*   **Max Segments:** 256 (due to 8-bit depth) [S1, unofficial].
*   **segmentEnabled:** Boolean flag to toggle segment-based styling [S2, unofficial].
*   **renderStyle:** Array of `X3DVolumeRenderStyleNode` objects. If empty, defaults are used [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Fractional Segmentation:** This method cannot represent voxels belonging to multiple segments simultaneously. Supporting fractional segmentation would require more memory and increase access complexity [S1, unofficial].
*   **Texture Component Ambiguity:** If a 3-component texture is used, the Red and Green components are repurposed for 2D data, which may conflict with standard RGB color expectations [S2, unofficial].
*   **Boundary Conditions:** The method assumes the 3D texture dimensions match the volume dimensions. Mismatches require interpolation or clamping, which is not specified [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_segmented_voxel(x, y, z, volume_data, segmented_voxels_texture, render_style_array, segment_enabled):
    """
    Renders a single voxel using 8-bit segment ordinal storage.
    
    Args:
        x, y, z: Voxel coordinates.
        volume_data: The underlying volumetric data (e.g., scalar values).
        segmented_voxels_texture: 3D texture containing segment IDs.
        render_style_array: List of rendering style objects.
        segment_enabled: Boolean flag.
        
    Returns:
        Rendered color/opacity for the voxel.
    """
    # Step 1: Retrieve segment ID from the 8-bit texture
    # Note: Implementation details for 2/3/4 component textures vary.
    # Assuming 4-component (RGBA) where Alpha is ID, or 3-component (RGB) where Blue is ID.
    # For generality, we assume a helper get_segment_id exists that handles component mapping.
    segment_id = get_segment_id_from_texture(segmented_voxels_texture, x, y, z)
    
    # Step 2: Check if segmentation is active
    if not segment_enabled:
        # Fallback to default rendering without segment-specific style
        style = get_default_style()
    else:
        # Step 3: Look up style based on segment ID
        # If render_style_array is empty, use default
        if not render_style_array:
            style = get_default_style()
        else:
            # Clamp segment_id to valid range [0, len(render_style_array)-1] to prevent out-of-bounds
            # [G] Guard against segment_id > 255 or index out of bounds
            index = min(segment_id, len(render_style_array) - 1)
            style = render_style_array[index]
            
    # Step 4: Apply style to volume data
    # The style object contains parameters like opacity map, color map, etc.
    rendered_value = apply_style(style, volume_data[x, y, z])
    
    return rendered_value

def get_segment_id_from_texture(texture, x, y, z):
    """
    Extracts the segment ID from the texture based on component count.
    [G] Placeholder logic for component mapping as per X3D spec notes.
    """
    rgba = texture.sample(x, y, z)
    if texture.num_components == 4:
        return int(rgba.a * 255) # Alpha channel
    elif texture.num_components == 3:
        return int(rgba.b * 255) # Blue channel
    elif texture.num_components == 2:
        # Not fully specified in sources; assume first component or specific mapping
        return int(rgba.g * 255) 
    else:
        return 0
```

## Subtopics

### Sparse structures
Use sparse data structures (e.g., octrees) for volumes with large empty regions. Choose when coverage is <33% to save memory, whereas 8-bit segment ordinal storage is better for dense segments. [Sparse structures](5-sparse-structures.md)

### Deferred Filtering Architecture for Compressed Volumes
Focuses on GPU decompression and filtering. 8-bit segment ordinal storage is a data representation method, not a decompression architecture. [Deferred Filtering Architecture for Compressed Volumes](857-deferred-filtering-architecture-for-compressed-volumes.md)

### S3Dc GPU Decompression and Rendering
A specific GPU-based compression scheme. 8-bit segment ordinal storage is a simpler, non-compressed (or lightly compressed) ordinal map.(1190-s3dc-gpu-decompression-and-rendering.md)

## Operating characteristics
*   **Time Complexity:** $O(1)$ per voxel for segment ID lookup and style application. No complex decoding required.
*   **Memory Complexity:** $O(W \times H \times D)$ bytes for the segment ordinal volume. This is significantly less than storing explicit $(x,y,z)$ coordinate lists for each segment if the segmentation covers $\ge 33\%$ of the volume [S1, unofficial].
*   **Performance:** Fast random access allows efficient ray marching. No decompression overhead compared to entropy-coded formats.
*   **Strengths:** Simple implementation, random access, low memory for dense segments, supports per-segment rendering styles.
*   **Limitations:** Limited to 256 segments. Cannot represent fractional/multi-label voxels. Less efficient than sparse structures for very sparse data.
*   **When to use:** When the volume has distinct, non-overlapping segments and rendering styles need to vary per segment. Ideal for medical segmentation visualization where regions are clearly defined.
*   **When not to use:** When fractional segmentation is required, or when the number of segments exceeds 255. When memory is extremely constrained and coverage is low (<33%).

## Background
The method originates from research in volume visualization, specifically in the context of non-photorealistic illustration and medical image rendering. It was formalized in the X3D standard for volume rendering, where it is known as `SegmentedVolumeData` [S2, unofficial]. The concept of using a "shadow volume" to record segment ordinals was proposed as an efficient alternative to explicit coordinate lists for segmentations with high voxel coverage [S1, unofficial].

## Sources
[S1] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA469447: Development of an Amendment to X3D to Create a Standard Specification of Medical Image Volume Rendering, Segmentation, and Registration, https://archive.org/details/DTIC_ADA469447, fetched 2026-10-26 (unofficial)
