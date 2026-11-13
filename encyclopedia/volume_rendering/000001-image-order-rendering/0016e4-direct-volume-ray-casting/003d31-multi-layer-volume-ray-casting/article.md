# Multi-layer Volume Ray Casting

Also called: raycast1Layer, segment-by-segment ray casting

## Overview
Multi-layer volume ray casting is a unified rendering framework that treats volume datasets and geometric objects (clipping geometry, embedded opaque/semi-transparent objects, editing masks, MPRs, and non-empty-space boundaries) as Volume-of-Interests (VOIs) [S1]. It solves the problem of handling spatial overlaps, empty-space skipping, and multiple material-specific transfer functions in a single pass by encoding VOI interactions into a pre-generated ray-layer buffer [S1]. Unlike standard single-layer ray casting, which cannot skip empty spaces between visible samples or handle embedded objects producing multiple layers, this method explicitly stores non-overlapping ray segments per pixel [S1]. It differs from polygon-assisted ray casting and single-pass ray casters that store full 3D texture coordinates by storing only the Z-coordinate of the starting position, reducing memory usage and ensuring sampling alignment across bricks [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset partitioned into bricks, a set of VOIs (each with a priority number and material ID), and a pre-generated ray-layer buffer.
*   **Output:** The final composited image color and opacity.

**Data Structures:**
*   **Ray-Layer Buffer:** A per-pixel array of ray segments [S1]. Each segment stores:
    *   `rayStart`: The starting Z-coordinate (depth) of the segment [S1].
    *   `rayEnd`: The ending Z-coordinate of the segment [S1].
    *   `materialID`: An integer identifying the material properties (e.g., transfer function) for this segment [S1].
    *   *Note:* The buffer encodes a set of non-overlapping ray segments for each pixel [S1].

**Parameters:**
*   `renderMpr`: A boolean flag indicating whether Multi-Planar Reformation (MPR) layers should be rendered before standard ray casting for the current segment [S1].
*   `VOI Priority`: An integer assigned to each VOI. Higher priority overwrites lower priority in overlapped regions. Multiple VOIs with identical priorities are blended [S1].
*   `Default VOI`: The implicit bounding box of the volume dataset, assigned the lowest priority. Voxels excluded by all other VOIs are rendered according to this default [S1].

**Algorithm Steps:**

1.  **Ray-Layer Buffer Generation (Pre-processing):**
    *   Convert all VOI information into a ray-layer buffer [S1].
    *   Perform depth peeling on all VOI boundaries [S1].
    *   Organize peeled fragments into layers based on VOI priority and material ID [S1].
    *   For each pixel, encode a set of non-overlapping ray segments, recording `rayStart`, `rayEnd`, and `materialID` [S1].
    *   *Note:* This step shifts computational workload to layer sorting to keep the ray-casting loop simple [S1].

2.  **Multi-Layer Ray Casting Loop:**
    *   For each pixel, fetch and decode the current layer information from the ray-layer buffer, including `rayStart`, `rayEnd`, and `materialID` [S1].
    *   If the segment is valid, call `raycast1Layer` with these parameters [S1].
    *   `raycast1Layer` returns a Boolean flag:
        *   `true`: Continue to the next layer in the buffer.
        *   `false`: Terminate the ray for the current brick [S1].

3.  **Single Layer Ray Casting (`raycast1Layer`):**
    *   **Brick Boundary Check:** Check if `rayStart` is behind the far end of the current brick. If so, return `false` to terminate the ray for the current brick [S1].
    *   **MPR Rendering:** If `renderMpr` is set, render the MPR layer in the intersection of `[rayStart, rayEnd]` with the current brick before standard ray casting [S1].
    *   **Standard Ray Casting:**
        *   Cast rays segment-by-segment within `[rayStart, rayEnd]` using the `materialID` to select the appropriate transfer function and viewing parameters [S1].
        *   Transfer functions are stacked in a single 2D texture. The texture coordinate offset is calculated as `materialID * D`, where `D` is the size of a single transfer function [S1].
        *   Use the ray-layer buffer to estimate gradients on layer surfaces to improve shading effects [S1].
    *   **Opacity Check:** Inspect accumulated opacity to determine possible early termination [S1].
    *   **Termination Check:** Check if `rayEnd` is behind the far end of the brick.
        *   If yes, return `false` (termination).
        *   Otherwise, return `true` to continue to the next ray segment [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_layer_volume_ray_casting(volume_data, ray_layer_buffer, bricks):
    """
    Main entry point for multi-layer volume ray casting.
    volume_data: 3D volume dataset
    ray_layer_buffer: 2D array where each pixel contains a list of segments
    bricks: List of brick objects, each with start_z and end_z
    """
    for pixel_x, pixel_y in image_dimensions:
        segments = ray_layer_buffer[pixel_x][pixel_y]
        accumulated_color = (0.0, 0.0, 0.0)
        accumulated_opacity = 0.0
        
        for segment in segments:
            ray_start = segment.start_z
            ray_end = segment.end_z
            material_id = segment.material_id
            
            # Find the brick containing this segment
            brick = find_brick_containing_z(bricks, ray_start)
            if brick is None:
                continue
                
            # Call single layer caster
            terminated = raycast1_layer(
                volume_data, 
                brick, 
                ray_start, 
                ray_end, 
                material_id,
                accumulated_color, 
                accumulated_opacity
            )
            
            if terminated:
                break
                
        # Composite final color for pixel
        image[pixel_x][pixel_y] = accumulated_color

def raycast1_layer(volume_data, brick, ray_start, ray_end, material_id, acc_color, acc_opacity):
    """
    Renders a single ray segment.
    Returns True if ray casting should terminate, False to continue.
    """
    # Check if rayStart is behind the far end of the current brick
    if ray_start >= brick.end_z:
        return True
        
    # Optional: Render MPR layer if flag is set
    if renderMpr_flag:
        render_mpr_layer(volume_data, brick, ray_start, ray_end)
        
    # Perform standard ray casting for the segment
    # This involves sampling, transfer function lookup (using material_id),
    # shading, and front-to-back compositing
    # See [Parent](19-direct-volume-ray-casting.md) for standard DVR steps
    
    # Inspect accumulated opacity for early termination
    if acc_opacity >= 1.0:
        return True
        
    # Check if rayEnd is behind the far end of the brick
    if ray_end >= brick.end_z:
        return True
        
    return False
```

**Edge Cases and Pitfalls:**
*   **Empty Spaces:** Single-layer ray casting cannot skip empty spaces between the first and last visible samples. Multi-layer encoding solves this by explicitly storing non-overlapping segments [S1].
*   **Embedded Objects:** Single-layer representation cannot cope with embedded objects that produce multiple layers. The ray-layer buffer explicitly handles these [S1].
*   **Bricking Compatibility:** Storing only the Z value of the starting position ensures sampling locations align across bricks, which is required for bricking [S1].

## Subtopics

### [Ray-Layer Buffer Generation via Depth Peeling](9474-ray-layer-buffer-generation-via-depth-peeling.md)
Generates the ray-layer buffer by depth-peeling VOI definitions and grouping fragments into ordered layers based on VOI priority and material ID. It differs from standard GPU-based CSG by performing stencil-like tests inside a GPU program, preserving priority and ID information.

### [Layer Separation Heuristic](11327-layer-separation-heuristic.md)
Determines optimal split points along a ray to separate high-quality raycasting from lower-cost image warping. It identifies specific volumetric features for light attenuation, preserving depth cues better than simple depth-based splitting.

## Operating characteristics
*   **Time Complexity:** The multi-pass depth peeling cost is nearly ignorable compared to the ray casting cost [S1]. The algorithm shifts computational workload to layer sorting to keep the ray-casting loop as simple as possible [S1].
*   **Memory Complexity:** The ray-layer buffer stores only the Z value of the starting position and material ID per segment, which is less memory-intensive than storing full 3D texture coordinates [S1].
*   **Performance:** Using a single shared ray-layer buffer for the whole volume (with bricking) is more efficient and easier to debug than using a different ray-layer buffer for each brick [S1].
*   **Strengths:** Supports interactive volume editing, cutaway views, and prioritized overlay of geometric masks, MPRs, and embedded objects [S1]. Handles multiple transfer functions in a single pass by stacking them in a 2D texture [S1].
*   **Limitations:** Requires pre-generation of the ray-layer buffer, which involves depth peeling [S1].

## Background
Multi-layer volume ray casting was presented by Wei Li from Siemens Corporate Research [S1]. It integrates various volume rendering enhancements into a unified framework, including empty-space skipping, volume clipping, embedding opaque and semi-transparent objects, and volume editing through CSG operations [S1]. The method treats all these objects as VOIs, assigning priority numbers to resolve spatial overlaps [S1].

## Sources
[S1] Multi-layer Volume Ray Casting on GPU, https://doi.org/10.2312/vg/vg10/005-012, fetched 2026-10-26
