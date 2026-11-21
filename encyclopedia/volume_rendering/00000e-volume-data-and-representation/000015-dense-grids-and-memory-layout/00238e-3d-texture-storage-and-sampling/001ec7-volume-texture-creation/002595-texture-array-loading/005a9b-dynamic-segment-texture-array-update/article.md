# Dynamic Segment Texture Array Update

Also called: Segment Texture Array Update

## Overview
Dynamic Segment Texture Array Update is a method for synchronizing GPU-side volumetric data with CPU-side segment definitions by rebuilding a texture array bound to shader uniforms. It solves the problem of efficiently mapping discrete, sparse voxel segments to a contiguous 3D texture array for rendering. It differs from standard volume rendering by treating the volume texture as a collection of independent 2D slices or bricks that are dynamically assembled based on a directory structure, rather than as a monolithic 3D buffer [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A directory texture containing segment metadata (X, Y, Z coordinates) and a collection of existing voxel textures indexed by segment ID.
*   **Output:** A rebuilt texture array bound to the shader uniform `segment_bricks`, along with updated uniforms for `active_segment_count` and `segment_directory`.

**Data Structures:**
*   **Directory Entry:** A texture or buffer structure containing three components (X, Y, Z). The source code implies these are unsigned integers or shorts, packed into a single entry.
*   **Segment ID:** A 32-bit integer computed from the directory entry components.
*   **Texture Array:** A `GL_TEXTURE_ARRAY` or equivalent GPU resource holding the voxel textures.

**Numbered Steps:**
1.  **Update Active Segment Count:** Set the shader uniform `active_segment_count` to the current number of active segments in the scene.
2.  **Update Directory Texture:** If a directory texture exists, set the shader uniform `segment_directory` to reference it. This allows the shader to access the mapping between logical segments and physical texture indices.
3.  **Rebuild Texture Array:** Iterate through the entries in the directory texture. For each entry:
    *   Compute the segment ID using the formula:
        $$ \text{segment\_id} = (\text{entry}.X \ll 18) \mid (\text{entry}.Y \ll 9) \mid \text{entry}.Z $$
        where `entry.X`, `entry.Y`, and `entry.Z` are treated as unsigned integers.
    *   Check if a valid texture exists for this computed `segment_id`.
    *   If the texture exists, add it to the texture array at the corresponding index.
4.  **Bind Texture Array:** Set the shader uniform `segment_bricks` to the newly rebuilt texture array.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dynamic_segment_texture_array_update(directory_texture, active_segment_count, texture_pool):
    """
    Rebuilds the segment texture array for GPU rendering.
    
    Args:
        directory_texture: A texture or buffer containing segment directory entries.
        active_segment_count: The current number of active segments.
        texture_pool: A dictionary or array mapping segment_id to GPU texture objects.
        
    Returns:
        None (updates shader uniforms directly)
    """
    # Step 1: Update active segment count
    set_shader_uniform("active_segment_count", active_segment_count)
    
    # Step 2: Update directory texture
    if directory_texture is not None:
        set_shader_uniform("segment_directory", directory_texture)
    
    # Step 3: Rebuild texture array
    new_texture_array = []
    
    # Iterate through directory entries. 
    # Note: In a real implementation, this requires reading from the directory texture 
    # on CPU or using a compute shader. Here we assume CPU-side iteration for clarity.
    for entry in directory_texture.entries:
        # Compute segment ID from directory components
        # X, Y, Z are assumed to be unsigned integers from the directory entry
        segment_id = ((entry.X << 18) | (entry.Y << 9) | entry.Z)
        
        # Check if texture exists for this segment ID
        if segment_id in texture_pool:
            new_texture_array.append(texture_pool[segment_id])
    
    # Step 4: Bind texture array
    # The texture array is created from the list of textures
    rebuilt_array = create_texture_array(new_texture_array)
    set_shader_uniform("segment_bricks", rebuilt_array)
```

**Parameters:**
*   `active_segment_count`: Integer, typically equal to the number of non-empty segments.
*   `segment_directory`: Texture reference, format depends on implementation (e.g., `RGBA32UI`).
*   `segment_bricks`: Texture array reference, format matches individual voxel textures (e.g., `RGBA8`).

**Edge Cases and Pitfalls:**
*   **Precision:** The bit-shifting formula assumes `entry.X` fits in 14 bits (since $2^{18}$ is the shift), `entry.Y` fits in 9 bits, and `entry.Z` fits in 9 bits. If coordinates exceed these ranges, the ID computation will collide. [S1, unofficial]
*   **Boundaries:** The source code checks for `ushort` limits in voxel space, but the directory texture format must align with the bit-packing assumptions. [S1, unofficial]
*   **Degenerate Input:** If `directory_texture` is null, the method should skip Step 2 and 3, or handle an empty array gracefully. [S1, unofficial]

## Subtopics

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of segments in the directory. The iteration and texture array rebuild are linear in the number of active segments.
*   **Memory Complexity:** $O(M)$ where $M$ is the number of active segments, as the texture array stores one texture per segment.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales linearly with the number of segments. Performance may degrade if the number of segments is very large due to texture array binding overhead.
*   **Strengths:** Efficiently handles sparse volumes by only including active segments in the texture array. Allows dynamic updates without rebuilding the entire volume texture.
*   **Limitations:** Requires CPU-side iteration to rebuild the texture array, which can be a bottleneck if done frequently. The bit-packing formula limits the coordinate space.
*   **When to use:** When rendering sparse voxel volumes with dynamic updates, such as in voxel-based games or simulations.
*   **When not to use:** For dense volumes where most segments are active, or when the coordinate space exceeds the bit-packing limits.

## Background
The method is derived from the Voxel2Pixel graphics library by Ben McLean, which implements voxel rendering techniques. The specific implementation details are found in the library's source code, particularly in the `SetupShaderUniforms` and related methods. [S1, unofficial]

## Sources
[S1] GitHub - BenMcLean/Voxel2Pixel: Graphics library for rendering voxel ..., https://github.com/BenMcLean/Voxel2Pixel, fetched 2026-10-29 (unofficial)
