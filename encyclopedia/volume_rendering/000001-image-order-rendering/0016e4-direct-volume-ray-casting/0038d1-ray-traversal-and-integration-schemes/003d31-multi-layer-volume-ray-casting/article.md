# Multi-layer Volume Ray Casting

Also called: raycast1Layer, segment-by-segment ray casting

## Overview
Multi-layer volume ray casting is a unified rendering framework that treats volume datasets and geometric objects (clipping geometry, embedded opaque/semi-transparent objects, editing masks, MPRs, non-empty-space boundaries) as Volume-of-Interests (VOIs) [S1]. It solves the problem of rendering complex, overlapping spatial regions with different material properties and priorities by encoding all VOI information into a pre-generated **ray-layer buffer** [S1]. Unlike standard ray marching which processes a single continuous ray or polygon-assisted methods that store full 3D coordinates, this method iterates through non-overlapping ray segments stored in the buffer, allowing material-specific transfer functions and shading per segment while maintaining bricking compatibility [S1].

## How it works
The method consists of two main phases: generating the ray-layer buffer via depth peeling, and performing segment-by-segment ray casting using that buffer [S1].

### Inputs and Outputs
**Inputs:**
*   Volume dataset (potentially bricked).
*   Set of VOIs, each defined by geometry or volume properties, a **Material ID**, and a **Priority Number** [S1].
*   Transfer functions, one per Material ID [S1].
*   `renderMpr` flag: Boolean indicating whether Multi-Planar Reformation (MPR) layers should be rendered [S1].

**Outputs:**
*   Final image.
*   Ray-layer buffer (intermediate data structure) [S1].

### Data Structures

**Ray-Layer Buffer**
*   **Layout:** A 2D texture (one entry per pixel).
*   **Content:** An array of ray segments.
*   **Segment Fields:**
    *   `rayStart`: The starting Z-coordinate (depth) of the segment within the current brick. Note: Only the Z value is stored, not full 3D texture coordinates [S1].
    *   `rayEnd`: The ending Z-coordinate of the segment [S1].
    *   `materialID`: An integer identifying the material properties (transfer function, shading parameters) for this segment [S1].
*   **Properties:** Segments are non-overlapping and ordered [S1].

**Transfer Function Storage**
*   Transfer functions for all material IDs are stacked into a single 2D texture [S1].
*   To sample the transfer function for a specific `materialID`, the texture coordinate offset is calculated as `materialID * D`, where `D` is the size (width or height, depending on layout) of a single transfer function in the texture [S1].

### Algorithm Steps

#### Phase 1: Ray-Layer Buffer Generation
1.  **VOI Definition:** Define all VOIs. Assign each a priority number (integer). Higher priority overwrites lower priority in overlaps. If priorities are identical, contributions are blended [S1].
2.  **Default VOI:** Assign the bounding box of the volume dataset as the default VOI with the lowest priority. Voxels excluded by all other VOIs are rendered according to this default [S1].
3.  **Depth Peeling:** Perform depth peeling on the boundaries of all VOIs [S1]. This is done via stencil-like tests inside a GPU program to preserve priority and ID information, rather than using hardware stencil buffers [S1].
4.  **Layer Simplification & Encoding:** Organize the peeled fragments into layers. For each pixel, encode a set of non-overlapping ray segments. Record the starting Z-coordinate (`rayStart`) and the `materialID` for each segment [S1].
5.  **Buffer Creation:** Store these segments in the ray-layer buffer. A single shared buffer is used for the whole volume (across all bricks) to ensure efficiency and debuggability [S1].

#### Phase 2: Multi-Layer Ray Casting
For each pixel, the ray caster iterates through the segments in the ray-layer buffer.

1.  **Fetch Segment:** Retrieve the current segment's `rayStart`, `rayEnd`, and `materialID` from the ray-layer buffer [S1].
2.  **Loop Condition:** If there are no more segments, terminate the ray for this pixel [S1].
3.  **Invoke Segment Renderer:** Call `raycast1Layer(rayStart, rayEnd, materialID, brickBounds)` [S1].
    *   *See Pseudocode below for `raycast1Layer` logic.*
4.  **Check Return Value:**
    *   If `raycast1Layer` returns `true`: Continue to the next segment in the buffer.
    *   If `raycast1Layer` returns `false`: Terminate the ray for this pixel (accumulated opacity is sufficient or ray is out of bounds) [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_layer_volume_casting(pixel_coords, ray_layer_buffer, bricks, renderMpr_flag):
    """
    Main loop for a single pixel.
    pixel_coords: (x, y)
    ray_layer_buffer: 2D texture containing arrays of segments per pixel
    bricks: List of brick objects, each with bounds and volume data
    renderMpr_flag: Boolean
    """
    # 1. Fetch the array of segments for this pixel from the buffer
    segments = ray_layer_buffer[pixel_coords]
    
    # 2. Initialize accumulation variables
    accumulated_color = (0.0, 0.0, 0.0, 0.0)
    accumulated_opacity = 0.0
    
    # 3. Iterate through segments
    for segment in segments:
        ray_start_z = segment.rayStart
        ray_end_z = segment.rayEnd
        material_id = segment.materialID
        
        # 4. Find the brick containing this segment
        # Note: In a bricked implementation, the ray-layer buffer is global,
        # but casting happens per brick. The segment's Z is relative to the brick.
        brick = get_brick_containing_z(ray_start_z)
        
        # 5. Call the segment renderer
        continue_casting = raycast1_layer(
            ray_start_z, 
            ray_end_z, 
            material_id, 
            brick, 
            renderMpr_flag
        )
        
        # 6. Update accumulation and check termination
        # Note: The specific accumulation logic (front-to-back) is assumed 
        # to be handled inside raycast1_layer or the caller. 
        # Based on [S1], raycast1_layer returns a boolean indicating if 
        # we should continue to the NEXT layer.
        
        if not continue_casting:
            break
            
    return accumulated_color

def raycast1_layer(ray_start_z, ray_end_z, material_id, brick, renderMpr_flag):
    """
    Renders a single ray segment.
    Returns True if the ray should continue to the next segment, False if terminated.
    """
    
    # Check if rayStart is behind the far end of the current brick
    if ray_start_z > brick.far_end_z:
        return False  # Terminate ray for this brick
    
    # If renderMpr flag is set, render MPR layer in the intersection
    if renderMpr_flag:
        mpr_intersection_start = max(ray_start_z, brick.near_end_z)
        mpr_intersection_end = min(ray_end_z, brick.far_end_z)
        if mpr_intersection_start < mpr_intersection_end:
            render_mpr_layer(mpr_intersection_start, mpr_intersection_end, material_id)
            # Note: [S1] implies MPR rendering happens before standard ray casting
            # for the segment. Accumulation logic for MPR is not detailed, 
            # but it contributes to the frame buffer.
    
    # Perform standard volume ray casting for the segment [rayStart, rayEnd]
    # within the brick, using the transfer function for material_id.
    # This step updates accumulated_color and accumulated_opacity.
    cast_volume_segment(ray_start_z, ray_end_z, material_id, brick)
    
    # Inspect accumulated opacity to determine possible termination
    if accumulated_opacity >= 1.0:
        return False  # Opaque, no need to process further layers
    
    # Check if rayEnd is behind the far end of the brick
    if ray_end_z > brick.far_end_z:
        return False  # All following layers are outside this brick
    
    return True  # Continue to next ray segment

def cast_volume_segment(ray_start_z, ray_end_z, material_id, brick):
    """
    Standard front-to-back ray casting for a segment.
    """
    step_size = get_step_size()
    current_z = ray_start_z
    
    while current_z < ray_end_z:
        # Sample volume data
        voxel_value = sample_volume(brick, current_z)
        
        # Get transfer function for material_id
        # Offset texture coordinate by material_id * D
        tf_offset = material_id * D
        color, opacity = sample_transfer_function(voxel_value, tf_offset)
        
        # Front-to-back compositing
        # C_out = C_in + C_seg * (1 - A_in)
        # A_out = A_in + A_seg * (1 - A_in)
        alpha_contribution = opacity * (1.0 - accumulated_opacity)
        accumulated_color += color * alpha_contribution
        accumulated_opacity += alpha_contribution
        
        current_z += step_size

def render_mpr_layer(start_z, end_z, material_id):
    """
    Renders an MPR layer.
    """
    # Implementation details for MPR rendering are not specified in [S1]
    # beyond the fact that it occurs in the intersection range.
    pass
```

### Parameters
*   **`renderMpr`**: Boolean flag. If true, MPR layers are rendered before standard ray casting for the current segment [S1].
*   **VOI Priority Number**: Integer assigned to each VOI. Higher priority overwrites lower priority. Identical priorities result in blending [S1].
*   **Default VOI**: Implicit bounding box of the volume dataset, lowest priority [S1].
*   **D**: Size of a single transfer function in the stacked 2D texture [S1].

### Edge Cases and Pitfalls
*   **Empty Space Skipping:** Single-layer ray casting cannot skip empty spaces between the first and last visible samples. The multi-layer buffer explicitly stores non-overlapping segments, solving this [S1].
*   **Embedded Objects:** Single-layer representations cannot cope with embedded objects producing multiple layers. The ray-layer buffer handles this by storing multiple segments per pixel [S1].
*   **Bricking Compatibility:** Storing only the Z value (not full 3D coordinates) ensures sampling locations align across bricks, which is critical for bricking [S1].
*   **Priority Conflicts:** If multiple VOIs overlap with different priorities, the higher priority VOI's material ID is used for the segment. If priorities are identical, blending occurs [S1].

## Subtopics

### [Ray-Layer Buffer Generation via Depth Peeling](9474-ray-layer-buffer-generation-via-depth-peeling.md)
Generates the ray-layer buffer by depth-peeling VOI definitions and grouping fragments into ordered layers based on VOI priority and material ID. It differs from standard GPU-based CSG by performing stencil-like tests inside a GPU program, preserving priority/ID info. Choose this when you need to handle complex VOI overlaps with priorities and material-specific rendering in a unified framework.

## Operating characteristics
*   **Time Complexity:** The multi-pass depth peeling cost is nearly ignorable compared to the ray casting cost [S1]. The algorithm shifts computational workload to layer sorting to keep the ray-casting loop simple [S1].
*   **Memory Complexity:** The ray-layer buffer stores only Z coordinates and material IDs per segment, reducing memory usage compared to methods storing full 3D texture coordinates [S1].
*   **Performance:** Using a single shared ray-layer buffer for the whole volume (with bricking) is more efficient and easier to debug than using a different buffer per brick [S1].
*   **Strengths:**
    *   Unified framework for VOIs, clipping, embedding, and CSG [S1].
    *   Supports interactive editing and cutaway views [S1].
    *   GPU-efficient due to simple ray-casting loop [S1].
    *   Enables gradient estimation on layer surfaces for improved shading [S1].
*   **Limitations:**
    *   Requires pre-generation of the ray-layer buffer (depth peeling) [S1].
    *   Relies on the assumption that depth peeling cost is negligible (valid for most scenes, but may vary) [S1].
*   **When to Use:** Medical imaging and scientific visualization software requiring interactive volume editing, cutaway views, and prioritized overlay of geometric masks, MPRs, and embedded objects [S1].
*   **When Not to Use:** Simple volume rendering without complex VOI interactions or embedded objects may not benefit from the overhead of buffer generation [S1].

## Background
Multi-layer volume ray casting was presented by Wei Li from Siemens Corporate Research [S1]. The paper was published in the proceedings of *Volume Graphics 2010* (VG10) [S1]. The method integrates various volume rendering enhancements (empty-space skipping, clipping, embedding, CSG) into a unified framework by treating them as VOIs [S1]. It builds on concepts from polygon-assisted ray casting and depth peeling but distinguishes itself by storing only Z coordinates and handling VOI priorities [S1].

## Sources
[S1] Multi-layer Volume Ray Casting on GPU, https://doi.org/10.2312/vg/vg10/005-012, fetched 2026-10-26
