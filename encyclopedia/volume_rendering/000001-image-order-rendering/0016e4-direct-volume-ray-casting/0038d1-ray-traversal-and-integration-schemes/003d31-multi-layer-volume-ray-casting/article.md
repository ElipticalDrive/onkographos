# Multi-layer Volume Ray Casting

Also called: raycast1Layer, segment-by-segment ray casting

## Overview
Multi-layer volume ray casting is a GPU-accelerated rendering method that handles complex volume-of-interest (VOI) interactions, including clipping, embedding, and CSG operations, by encoding spatial overlaps into a pre-generated ray-layer buffer [S1]. Unlike standard ray casting which processes a single continuous segment or requires multiple passes for layered objects, this method iterates through non-overlapping ray segments stored in the buffer, applying material-specific transfer functions and priorities per segment within a single rendering pass [S1]. It sets itself apart from polygon-assisted ray casting by storing only the Z-coordinate of layer starts rather than full 3D texture coordinates, reducing memory usage and ensuring bricking compatibility [S1].

## How it works
The method operates in two distinct phases: the generation of a ray-layer buffer that encodes VOI priorities and boundaries, and the segment-by-segment ray casting loop that renders these layers [S1].

### Inputs and Outputs
**Inputs:**
*   Volume dataset (scalar field).
*   Set of Volume-of-Interests (VOIs), each defined by geometry (e.g., clipping planes, embedded objects, MPRs) and assigned a priority integer and a material ID [S1].
*   Transfer functions mapped to material IDs [S1].
*   `renderMpr` flag: Boolean indicating whether Multi-Planar Reformation (MPR) layers should be rendered before volume ray casting for the current segment [S1].

**Outputs:**
*   Final image (frame buffer) [S1].
*   Ray-layer buffer: A 2D texture where each pixel contains an array of ray segments [S1].

### Data Structures
**Ray-Layer Buffer Layout:**
The buffer is a per-pixel array of ray segments. Each segment stores:
1.  `rayStart`: The starting Z-coordinate (depth) of the segment within the current brick [S1].
2.  `rayEnd`: The ending Z-coordinate of the segment [S1].
3.  `materialID`: An integer identifying the material properties (transfer function, shading parameters) for this segment [S1].

The Z value represents the starting position of a new layer. This design reduces memory usage compared to storing full 3D texture coordinates and ensures sampling locations align across bricks for bricking compatibility [S1].

**Transfer Function Texture:**
To support multiple transfer functions in a single pass, they are stacked into a single 2D texture. The texture coordinate offset is calculated as `materialID * D`, where `D` is the size of a single transfer function in the texture [S1].

### Algorithm Steps

#### Phase 1: Ray-Layer Buffer Generation
1.  **VOI Definition:** Define all VOIs (volume data, clipping geometry, embedded objects, MPRs, non-empty-space boundaries). Assign each VOI a priority number (integer) and a material ID [S1].
    *   Higher priority VOIs overwrite lower priority ones in overlapping regions [S1].
    *   If multiple VOIs share identical priorities, their contributions are blended [S1].
    *   The default VOI is the implicit bounding box of the volume dataset with the lowest priority [S1].
2.  **Depth Peeling:** Perform depth peeling on all VOI boundaries [S1]. This is done inside a GPU program using stencil-like tests rather than hardware stencil buffers to preserve priority and ID information [S1].
3.  **Layer Simplification & Encoding:** Organize the peeled fragments into layers. For each pixel, encode a set of non-overlapping ray segments, recording the starting Z-location and material ID for each segment [S1].
    *   This step shifts computational workload to layer sorting to keep the ray-casting loop simple [S1].

#### Phase 2: Multi-Layer Ray Casting
The ray caster iterates through the ray-layer buffer. The process is repeated for each brick in a bricked volume, but the ray-layer buffer is shared across all bricks [S1].

1.  **Fetch Segment:** For the current pixel, fetch the current layer information from the ray-layer buffer: `rayStart`, `rayEnd`, and `materialID` [S1].
2.  **Check Validity:** If the segment is valid (non-empty), proceed. If no more segments exist, terminate the ray for this pixel [S1].
3.  **Call `raycast1Layer`:** Invoke the single-layer ray casting function with `rayStart`, `rayEnd`, and `materialID` [S1].
    *   `raycast1Layer` returns a Boolean flag: `true` to continue to the next layer, `false` to terminate the ray [S1].
4.  **Loop:** If `raycast1Layer` returns `true`, fetch the next segment from the buffer and repeat from Step 1. If `false`, terminate the ray for this pixel [S1].

**`raycast1Layer` Implementation Details:**
1.  **Brick Boundary Check:** Check if `rayStart` is behind the far end of the current brick. If so, return `false` (terminate ray for this brick) [S1].
2.  **MPR Rendering (Optional):** If `renderMpr` is set, render the MPR layer in the intersection of `[rayStart, rayEnd]` with the current brick before standard volume ray casting [S1].
3.  **Standard Ray Casting:** Perform front-to-back volume ray casting for the segment `[rayStart, rayEnd]` using the transfer function associated with `materialID` [S1].
    *   Use the ray-layer buffer to estimate gradients on layer surfaces to improve shading effects [S1].
4.  **Opacity Check:** Inspect the accumulated opacity. If it reaches 1 (or a threshold), early termination may occur [S1].
5.  **Exit Check:** Check if `rayEnd` is behind the far end of the brick.
    *   If yes, return `false` (all following layers are outside this brick, terminate) [S1].
    *   If no, return `true` (continue to the next ray segment in the buffer) [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_layer_volume_ray_casting(volume_data, bricks, ray_layer_buffer, transfer_functions, render_mpr_flag):
    """
    Main entry point for multi-layer volume ray casting.
    Iterates through bricks and pixels, processing segments from the ray-layer buffer.
    """
    for brick in bricks:
        # The ray-layer buffer is shared for the whole volume, but casting is done per brick
        for y in range(height):
            for x in range(width):
                # Get the list of segments for this pixel from the buffer
                segments = ray_layer_buffer.get_segments(x, y)
                
                accumulated_color = 0.0
                accumulated_opacity = 0.0
                
                for segment in segments:
                    ray_start = segment.start_z
                    ray_end = segment.end_z
                    material_id = segment.material_id
                    
                    # Call the single-layer caster
                    # Returns True if we should continue to next segment, False if terminated
                    should_continue = raycast1_layer(
                        volume_data, 
                        brick, 
                        ray_start, 
                        ray_end, 
                        material_id, 
                        transfer_functions, 
                        render_mpr_flag,
                        accumulated_color,
                        accumulated_opacity
                    )
                    
                    if not should_continue:
                        break
                
                # Write final accumulated_color to frame buffer
                frame_buffer.set_pixel(x, y, accumulated_color)

def raycast1_layer(volume_data, brick, ray_start, ray_end, material_id, transfer_functions, render_mpr_flag, current_color, current_opacity):
    """
    Renders a single ray segment.
    Returns True if the ray should continue to the next segment, False if terminated.
    """
    # 1. Check if rayStart is behind the far end of the current brick
    if ray_start >= brick.far_end_z:
        return False
    
    # 2. Optional MPR Rendering
    if render_mpr_flag:
        # Render MPR layer in the intersection of [ray_start, ray_end] with the brick
        # This modifies current_color and current_opacity
        render_mpr_layer(volume_data, brick, ray_start, ray_end, current_color, current_opacity)
        
        # Check if opacity is fully opaque after MPR
        if current_opacity >= 1.0:
            return False

    # 3. Standard Volume Ray Casting for the segment
    # Cast ray from ray_start to ray_end
    # Use transfer function indexed by material_id
    # Update current_color and current_opacity using front-to-back compositing
    
    # Estimate gradients from ray-layer buffer for shading
    gradients = estimate_gradients_from_buffer(ray_start, ray_end)
    
    t = ray_start
    while t < ray_end:
        # Sample volume data at t
        value = sample_volume(volume_data, t)
        
        # Get color and opacity from transfer function
        # Offset texture coordinate by material_id * D
        color, opacity = get_transfer_function(material_id, value)
        
        # Compositing
        # C' = C' + (1 - alpha') * alpha * C
        # alpha' = alpha' + (1 - alpha') * alpha
        current_color = current_color + (1.0 - current_opacity) * opacity * color
        current_opacity = current_opacity + (1.0 - current_opacity) * opacity
        
        t += step_size
        
        # Early termination if fully opaque
        if current_opacity >= 1.0:
            break

    # 4. Check if rayEnd is behind the far end of the brick
    if ray_end >= brick.far_end_z:
        return False
    
    return True
```

### Parameters
*   `renderMpr`: Boolean flag. Default is false. When true, MPR layers are rendered before volume ray casting for the segment [S1].
*   `VOI priority number`: Integer. Determines rendering order in overlapped regions. Higher priority overwrites lower priority. Multiple VOIs may share identical priorities, in which case their contributions are blended [S1].
*   `Default VOI`: Implicit bounding box of the volume dataset, assigned the lowest priority. Voxels excluded by all other VOIs are rendered according to this default [S1].

### Edge Cases and Pitfalls
*   **Empty Space Skipping:** Single-layer ray casting cannot skip empty spaces between the first and last visible samples. Multi-layer encoding explicitly stores non-overlapping segments, solving this [S1].
*   **Embedded Objects:** Single-layer representation cannot cope with embedded objects that produce multiple layers. The ray-layer buffer handles this by storing multiple segments per pixel [S1].
*   **Bricking Compatibility:** Storing only the Z value (not full 3D texture coordinates) ensures sampling locations align across bricks, which is critical for bricking [S1].
*   **Priority Conflicts:** If multiple VOIs with different viewing parameters have identical priorities, their contributions are blended. This requires careful handling in the transfer function application [S1].

## Subtopics

### [Ray-Layer Buffer Generation via Depth Peeling](9474-ray-layer-buffer-generation-via-depth-peeling.md)
This method generates the ray-layer buffer for multi-layer volume ray casting by depth-peeling Volume of Interest (VOI) definitions and grouping the resulting fragments into ordered layers based on VOI priority and material ID. It differs from standard GPU-based Constructive Solid Geometry (CSG) by performing stencil-like tests inside a GPU program rather than using hardware stencil buffers, thereby preserving critical priority and ID information required for correct layer placement and optimization.

## Operating characteristics
*   **Time Complexity:** The multi-pass depth peeling cost is nearly ignorable compared to the ray casting cost [S1]. The algorithm shifts computational workload to layer sorting to keep the ray-casting loop as simple as possible [S1].
*   **Memory Complexity:** The ray-layer buffer stores only the Z value of the starting position and material ID per segment, reducing memory usage compared to methods storing full 3D texture coordinates [S1].
*   **Performance:** Using a single shared ray-layer buffer for the whole volume (with bricking) is more efficient and easier to debug than using a different ray-layer buffer for each brick [S1]. Compared to multi-pass depth peeling on intersecting volumes that raycasts each layer in a separate pass, this method stores peeling information in a single ray-layer buffer, requiring only one raycasting pass, which is more efficient for GPU hardware [S1].
*   **Strengths:** Supports interactive changing of shape and appearance of VOIs; handles complex CSG operations, clipping, and embedding in a unified framework; efficient GPU implementation due to simple ray-casting loop [S1].
*   **Limitations:** Requires pre-generation of the ray-layer buffer; relies on depth peeling which may have overhead for very complex VOI configurations (though noted as ignorable compared to ray casting) [S1].
*   **When to Use:** Medical imaging and scientific visualization software for interactive volume editing, cutaway views, and prioritized overlay of geometric masks, MPRs, and embedded objects [S1].
*   **When Not to Use:** Simple volume rendering without complex VOI interactions or clipping; scenes where pre-computation of the ray-layer buffer is prohibitive.

## Background
Multi-layer volume ray casting was presented by Wei Li from Siemens Corporate Research in 2010 [S1]. It integrates various volume rendering enhancements into a unified framework, including empty-space skipping, volume clipping, embedding opaque and semi-transparent objects, and volume editing through CSG operations [S1]. The central idea is to treat all these objects as volume-of-interests (VOIs) and encode their information into a ray-layer buffer via depth peeling and layer simplification [S1]. This approach addresses limitations in previous methods such as polygon-assisted ray casting (which stores 3D texture coordinates) and multi-pass depth peeling (which requires separate rendering passes for each layer) [S1].

## Sources
[S1] Multi-layer Volume Ray Casting on GPU, https://doi.org/10.2312/vg/vg10/005-012, fetched 2026-10-26
